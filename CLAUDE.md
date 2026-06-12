# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

TAO Community Edition (TAO CE) — an open-source e-testing platform by Open Assessment Technologies. This is a **meta/orchestration repository**: the actual application source code lives in ~20 git submodules under `apps/*/src` (see `.gitmodules`, all on `tao-ce/main` branches). This repo provides the build toolchain, container packaging, systemd units, configuration templates, and docker-compose stacks that assemble those apps into a running product.

There are no tests or linters at this level — those belong to the individual app submodules.

## Development environment

Development happens **inside the devcontainer** (`.devcontainer/`, built from `docker-compose.dev.yml`). The devcontainer runs systemd and podman/buildah internally; apps are built as container images, their contents exported to `/opt/tao-ce/<app>`, and run as `tao-ce.<app>*.service` systemd units.

Prerequisites: ssh-agent with a GitHub key (most submodules are private), NPM token in `.secrets/npm`, submodules initialized (`git submodule update --init --recursive`).

## Common commands

All orchestration uses [go-task](https://taskfile.dev) (`Taskfile.yml`). Run from the devcontainer terminal:

```bash
task dev:init                    # initialize dev environment (setup/static services, logdy)
task dev:up                      # build all apps from source and start them (main entrypoint)

# Per-app lifecycle (app names: construct, deliver, portal, proctoring, scoring,
# timers, hierarchy, devkit, datastore, dynamic-query, environment-management (alias: em),
# task-orchestrator, content-service):
task apps:<app>:build            # build the app's container image with buildah
task apps:<app>:up               # build + deploy to /opt/tao-ce/<app> + (re)start systemd units
task apps:<app>:clean-start      # force restart on next start (clears start signal)

task dev:wipe                    # wipe all persistent data (ES, /var/lib/tao-ce, /etc/tao-ce/setup)
task dev:down                    # stop all tao-ce services and clean /opt/tao-ce
task dev:prune                   # remove build caches (buildah/podman reset)

# Release image (single distributable "swift" container, see docs/build/README.md):
task swift:build && task swift:tag    # requires CE_MINOR, CE_MAJOR, TAO_VERSION, REMOTE_REGISTRY env vars
task swift:login && task swift:push
```

Incremental builds are signal-file based (`.cache/build/signals`); tasks skip when sources are unchanged. App logs go to journald (each build/runtime line tagged with `APP=`/`TIER=`); follow deployment via the logdy UI at `http://localhost:18995`.

To just **run** the released product (no source build): `docker compose up` from the repo root pulls `quay.io/tao-ce/tao-ce` plus its dependency stack. Then add `0.0.0.0 community.tao.internal` to `/etc/hosts` and open `https://community.tao.internal/portal` (default login `admin` / `password`).

## Architecture

### Two build toolchains (`build/`)

- **`build/crystal/`** — "from sources" path used in the devcontainer. A shared parameterized Taskfile (included once per app in `apps/Taskfile.yml`, which defines each app's build contexts and base images). Flow per app: `buildah build` → image in local podman store → `deploy` exports the image filesystem to `/opt/tao-ce/<app>` via `podman image mount` + rsync → `start` enables/restarts the `tao-ce.<app>*.service` units from `<app>/meta/systemd/`.
- **`build/swift/`** — packages everything into the single `tao-ce` container image published to quay.io for end users.

### Apps (`apps/`)

Each app directory contains: `src/` (submodule(s) with real sources), a `Dockerfile`, and `meta/systemd/` with its service units. Apps are mixed-stack: PHP/composer (construct — the TAO core extensions), Node, and Go (proctoring, hierarchy, parts of environment-management). One app often maps to several systemd services (e.g. portal: backend, worker, init, bootstrap…).

### Dependency stack (`docker-compose.stack.yml`)

Shared infrastructure services, used by both the dev and release compose files: PostgreSQL 17, Valkey (Redis), Elasticsearch 8, a Google Pub/Sub emulator, and a Firestore emulator (the emulators are built from `services/`).

### Configuration and provisioning (`etc/`, `libexec/`)

- `etc/` — config files copied into images / the dev host (`task dev:sync:etc`).
- `libexec/setup/` — jsonnet templates that generate per-app config (run by `tao-ce.setup.service`); the runtime config entrypoint is `/etc/tao-ce/config/tao.yaml` (see the `configs:` block in `docker-compose.yml` for its schema: publicDomain, defaultLocale, dependency addresses).
- `libexec/init/` — initialization data; `libexec/pubsub/` — Python script provisioning Pub/Sub topics/subscriptions; `libexec/systemd/` — common setup/static units.

### `hack/`

Acknowledged temporary shortcuts (task and artifact management) intended to be replaced before/after public release — don't treat patterns here as canonical.

## Git workflow

Work lands on `develop` and is merged to `main` via PR for releases. Submodule pointers are updated with `chore: update submodule` commits.

## Further docs

- `INSTALL.md` — end-user install/usage walkthrough
- `docs/dev/` — devcontainer environment details and troubleshooting
- `docs/build/README.md` — release image build/publish procedure
- `docs/ops/` — domain, proxy, TLS, wipe/reinstall operations
- `docs/repos.md` — repository structure reference
