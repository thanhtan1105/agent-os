# Production VPS deployment

The `Deploy AgentOS to production VPS` workflow is a manual-only deployment
for the repository's self-hosted Linux runner. It builds the checked-out
revision with the repository `Dockerfile`, then starts the gateway through the
repository `compose.yaml`. The host port remains bound to `127.0.0.1` by the
compose file; expose it only through a reverse proxy or VPN with an appropriate
authentication policy.

## Required GitHub configuration

Create the `production` GitHub Environment and set the following Actions
variable (preferred) or Environment secret:

| Name | Purpose |
| --- | --- |
| `AGENTOS_DEPLOY_DIR` | Existing, writable, non-root absolute deployment directory. The workflow writes `releases/<commit-sha>` and the `current` symlink here. |

Environment secrets such as provider API keys remain in the VPS deployment
root's `.env` file; they are not copied from Actions and must never be added to
the repository. The workflow uses the Environment so those controls apply even
though it runs directly on the self-hosted runner.

The same `.env` must define `AGENTOS_AUTH_TOKEN`. Generate this once on the
VPS as `agentos` with `openssl rand -hex 32`, store the result only in `.env`,
and configure the reverse proxy or browser client to send it. Docker requires a
non-loopback listener inside its network namespace; token auth keeps the
loopback-only host port safe if its mapping changes.

Configure a required reviewer for the `production` Environment before allowing
anyone besides the repository owner to dispatch the workflow.

## VPS prerequisites

The self-hosted runner service account must be able to run commands as the
non-login `agentos` service user without an interactive password. Test that
from an administrator shell on the VPS:

```sh
DEPLOY_DIR="/set/the/configured/deployment/directory"
sudo -iu github-runner env DEPLOY_DIR="${DEPLOY_DIR}" bash -lc \
  'set -e; sudo -n -u agentos -- id -un; sudo -n -u agentos -- test -w "${DEPLOY_DIR}"; sudo -n -u agentos -- docker compose version'
```

The command must print `agentos`, complete with exit status 0, and must not
prompt for a password. It also requires Docker, Docker Compose v2, Git LFS,
tar, curl, and outbound access to fetch the source, Git LFS assets, and
container build dependencies.

The deployment root must be owned and writable by `agentos`. Keep long-lived
files such as `.env` at its root, not inside a release directory. Releases are
immutable directories and are deliberately not auto-deleted; that keeps a
previous build available for a deliberate rollback.

## Deploy and rollback

Dispatch **Deploy AgentOS to production VPS** from GitHub Actions, enter the
branch, tag, or full commit SHA to deploy, and wait for the health check at
`http://127.0.0.1:18791/healthz` to pass.

To roll back, point `current` at a prior release and restart the fixed Compose
project as the service user:

```sh
DEPLOY_DIR="/set/the/configured/deployment/directory"
sudo -u agentos ln -sfn "${DEPLOY_DIR}/releases/COMMIT_SHA" "${DEPLOY_DIR}/current"
sudo -u agentos docker compose --project-name agentos \
  --project-directory "${DEPLOY_DIR}" \
  -f "${DEPLOY_DIR}/current/compose.yaml" up -d gateway
```

Replace the placeholder path and commit SHA with an existing release. Do not
use `--remove-orphans` during a manual rollback unless you have confirmed that
the project owns every affected container.

---

[Docs index](README.md) · [Gateway operations](gateway.md)
