---
name: bring-up-cloud-stack
license: Apache-2.0
description: Use when starting Mission Control cloud services from the repository compose stack.
metadata:
  author: NVIDIA Isaac Team <info@nvidia.com>
  version: 1.0.0
  permissions:
    file_read:
      - resources/docker-compose/bringup_services.yaml
      - resources/docker-compose/.env
      - resources/docker-compose/images.conf
      - resources/app/config/defaults.yaml
      - references/bringup-reference.md
    file_write: []
    execute:
      - skills/bring-up-cloud-stack/scripts/preflight.sh
      - skills/bring-up-cloud-stack/scripts/up.sh
---

# Bring up the Mission Control cloud stack

Use the installed skill's `scripts/preflight.sh` and `scripts/up.sh`. The skill
bundles the compose file, environment, images and application configuration under
`resources/`; a separate product checkout and exported compose variables are not
required. Invoke scripts by absolute path when outside the repository.

## Bringup

1. Identify requested options: cloud only (default), simulator, pull, foreground,
   configuration reread, or dry run.
2. Run `skills/bring-up-cloud-stack/scripts/preflight.sh`. It is read-only. Resolve
   every FAIL before starting services. Docker Engine, Compose v2, NVIDIA GPU,
   driver/container runtime, setfacl, configuration and free host ports are required.
   Images missing locally are WARN: normal startup pulls them automatically.
3. Run `skills/bring-up-cloud-stack/scripts/up.sh` with the requested options. Use
   a script runner if available, otherwise invoke the script directly. It sources
   the resolved configuration and starts the selected profiles detached by default.
   Use this runner instead of assembling a normal `docker compose up` yourself.
4. After startup, check the configured Mission Control `/api/v1/health` endpoint
   and container status/logs. Read `MC_PORT` from the resolved `.env`; do not guess
   ports or release tags. Report readiness and any remaining failed services.

The default profile starts mosquitto, postgres, mission-database,
mission-dispatch, wpg, cuopt and mission-control. Wpg and cuopt need GPU access.
`--with-sim` adds the generic robot simulator; orchestration supplying a separate
robot runtime must not use it.

## Options

| Option | Meaning |
| --- | --- |
| `--with-sim` | Add the generic robot-simulator profile. |
| `--pull` | Explicitly pull selected images; use after image references change. |
| `--foreground` | Attach to startup/logs instead of default detached startup. |
| `--dry-run` | Preview commands without starting services. |
| `--recreate` | Force recreation; requires the safeguards below. |

Preflight accepts `--with-sim` and `--pull`; its pull checks remain offline.
The selected shipped service images are public; private replacements require the
user's registry credentials.

## Workspace and configuration

`MC_WORKSPACE` defaults to `./skills-workspace`. Each compose file, `.env`, and
`images.conf` resolves from that workspace if present, otherwise from the skill's
resources. Images are sourced after `.env`. The runner sets `CONFIG_DIR` to
`$MC_WORKSPACE/app/config`, copies only missing application config/map files and
ensures the compose file has its adjacent `init-db.sh`. It never overwrites user
configuration or writes back to resources. The scripts own variable preflight;
callers need not manually export the full variable list.

Use `change-map`, `change-fleet-composition` or `change-stack-version` for requested
changes. Keep MQTT transport and ports consistent with the selected compose
configuration; do not launch a separate Dispatch container to bypass it.

## Recovery and limits

- Before `--recreate`, show the exact `--recreate --dry-run` preview, obtain explicit
  confirmation, and confirm that no active robot missions are in progress.
- Preserve existing services and evidence. Do not remove containers, prune Docker,
  delete volumes, or use `docker compose down -v` as automatic recovery.
- Inspect logs when health fails. Successful service bringup does not prove a robot
  mission completed; use `submit-and-monitor-mission` for requested mission validation.
- Building images, Helm deployments, alternative demo compose stacks and isolated
  service subsets are outside this skill's normal bringup path.

Read [the detailed reference](references/bringup-reference.md) for conditional
configuration, troubleshooting, logs, stopping, or advanced flag details.
