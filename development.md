# Development and integration

Run Compose commands from `aess/docker/`.

```bash
docker compose -f compose.yaml -f compose.dev.yaml up -d --build --wait
docker compose -f compose.yaml -f compose.dev.yaml restart strong-node
# After editing a robot, rerun its job with the same override:
docker compose -f compose.yaml -f compose.dev.yaml --profile mission rm -f writer executor
docker compose -f compose.yaml -f compose.dev.yaml --profile mission up -d writer executor
```

The override mounts each project's source read-only. These are plain Python
scripts, so restart servers or rerun jobs after editing; there is no auto reload.
The base deployment uses baked sources. Add future Dockerfiles/Compose overrides
in this directory, using explicit source contexts and service-name DNS. Preserve
service names and volume identity unless intentionally performing a migration.
Do not add an analysis worker, queue, or database until application code needs it.

## Code responsibilities

Keep changes within their existing component repository. Run unit tests while
iterating, then `./docker/test.sh` from the workspace root for integration changes.
Use the [code structure proposal](code-structure.md) to migrate modules in small,
testable steps. The Python projects currently use flat modules and standard-library
imports; the proposed `src/` packages have not been implemented yet.

## Extending Docker integration

1. Put a new component Dockerfile in `docker/` and give it an explicit source build context.
2. Add only services the new application code actually requires.
3. Use Compose service names for container connections and declare environment inputs in `docker/.env.example`.
4. Add a health check for a long-running service, or use successful completion for a finite initialization/job dependency.
5. Give persistent data a named volume; keep internal ports unpublished.
6. Add integration assertions to the isolated test workflow, then document the actual service and its settings here.

The parent workspace has no Git repository. `Network/`, `Robots-Network/` and `web-app/` are
independent repositories. Shared `docs/`, `docker/` and the root README must be
versioned in a parent integration repository; component commits do not include
them. No parent repository or submodules have been created automatically.

Backend, network-bridge and executor-control have development source mounts.
Dashboard remains a static build and requires an image rebuild after source edits;
no host Node installation is needed. See [webapp.md](webapp.md).
