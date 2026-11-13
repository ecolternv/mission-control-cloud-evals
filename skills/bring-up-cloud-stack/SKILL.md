---
name: bring-up-cloud-stack
license: Apache-2.0
description: Use when starting Mission Control cloud services from the repository compose stack.
metadata:
  author: NVIDIA Isaac Team <info@nvidia.com>
  version: 1.0.0
  permissions:
    file_read:
      - references/bringup-reference.md
      - resources/docker-compose/bringup_services.yaml
      - resources/docker-compose/.env
      - resources/docker-compose/images.conf
      - resources/app/config/defaults.yaml
    file_write: []
    execute:
      - skills/bring-up-cloud-stack/scripts/preflight.sh
      - skills/bring-up-cloud-stack/scripts/up.sh
---

# Bring up cloud stack

Use the bundled scripts to start the prebuilt Mission Control cloud services.
The ordinary cloud-only, detached startup is fully described here. The scripts
resolve configuration and install missing default config; no source checkout,
repository search, or script audit is needed for this path.

## Default path

1. Use the installed skill directory (the parent of this file). Preserve
   `MC_WORKSPACE` and other supplied environment variables. If unset,
   `MC_WORKSPACE` defaults to `./skills-workspace` in the caller's directory.
   Workspace files override bundled `resources/` files path by path. The runner
   sources `.env`, then `images.conf`; it seeds missing workspace app config
   without overwriting user files. Do not manually export all compose variables.
2. Run preflight and startup in one shell execution, stopping if preflight fails:

   ```bash
   bash /absolute/path/to/bring-up-cloud-stack/scripts/preflight.sh &&
     bash /absolute/path/to/bring-up-cloud-stack/scripts/up.sh
   ```

   Substitute the actual installed skill path. Preflight checks Docker, Compose
   v2, configuration, GPU/runtime and ports. Resolve every `FAIL` before starting.
   Missing images are warnings: the runner pulls them during ordinary startup.
   Run the existing runner once and wait for that session to finish; preserve its
   exit status. Do not launch another runner merely to check progress.
3. After startup, use `docker ps -a` to inspect service state, and check
   `http://localhost:${MC_PORT}/api/v1/health`. Read only the needed `MC_PORT`
   value from the resolved `docker-compose/.env` (workspace override, otherwise
   `resources/`); do not dump credentials. Allow bounded startup settling time.
   Confirm all seven services are Up: `mosquitto`, `postgres`,
   `mission-database`, `mission-dispatch`, `wpg`, `cuopt`, `mission-control`.
   Successful runner exit or a healthy HTTP endpoint alone does not establish
   that every service started.
4. For an exited/unhealthy service, inspect its bounded log tail and dependencies,
   then diagnose the observed error. Only persistent or unfamiliar failures need
   the troubleshooting reference. Avoid collecting healthy-service logs or broad
   repository/configuration dumps without a diagnostic reason. Preserve failure
   details and report unresolved services. Once checks pass, report readiness,
   the health URL, and how to inspect a service with `docker logs --tail 80 NAME`.
   Readiness does not prove robot mission correctness; that requires
   `submit-and-monitor-mission`.

## Requested modes and safety

- Use `up.sh` for normal bringup, rather than constructing a compose command.
  `--with-sim` enables the generic simulator; omit it for orchestrations supplying
  their own robot runtime. `--foreground` attaches logs; `--dry-run` previews.
- Use `--pull` after requested image-reference changes. Pass `--with-sim` and
  `--pull` to preflight too when applicable. Read the relevant section of
  [the reference](references/bringup-reference.md) for nondefault modes, existing
  container/image conflicts, stopping, custom configuration, or persistent errors.
- Do not change images, maps, fleet, MQTT topology or credentials unless requested.
  Use the corresponding mutation skill for requested configuration edits.
- Never use `docker rm`, `docker system prune`, or `docker compose down -v` for
  recovery. `--recreate` interrupts services: require the exact dry-run preview,
  explicit user confirmation, and confirmation of no active robot missions first.
  Do not launch a separate Dispatch container as a workaround.
- Scope: the bundled compose cloud stack, not Helm, local image builds, demo
  variants, or arbitrary service subsets.

## Waiting without model polling rounds

When asynchronous tool orchestration is available, start the shell execution
once and keep its session waits inside one awaited code loop. While a session ID
exists, await that same session with empty input and the supported blocking wait;
retain the final output tail and emit exit status plus that tail on completion.
Keep full diagnostics available; do not suppress errors. Coordinate outer and
inner wait budgets: for example a supported 60-second outer execution can cover
repeated 30-second session waits. If the outer execution yields a cell ID, resume
that cell rather than creating another loop. Honor actual tool limits and user
progress-update requirements. Otherwise use the normal foreground/session waits.
Do the service-state and health checks above after the command completes.
