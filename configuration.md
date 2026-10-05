# Configuration

Run shell commands in this guide from `aess/docker/`. See the [run guide](run-guide.md) for first startup.

The active environment file is `docker/.env`; `Network/.env` is no longer used.
On a fresh checkout with no existing `.env`, generate a key as follows. If `.env`
already exists, keep it: these commands replace its contents and rotate the key.

```bash
# Generate the required secret using container Python:
docker run --rm python:3.13-slim-bookworm@sha256:5024f48ba9441d4b13a95d3945abc6365538e3a31109833367a1923523c6efed python -c 'import secrets; print("SHARED_SECRET=" + secrets.token_hex(32))' > .env
printf '%s\n' 'COMPOSE_FILE=compose.yaml:compose.ros.yaml' >> .env
chmod 600 .env
```

Both source build contexts ignore `.env` files and have explicit COPY rules. The
integration directory is outside the build contexts. Never commit private `.env`;
`docker/.gitignore` excludes it. Docker administrators can inspect environment
values, so restrict daemon access. Configure external clients with the same key.

| Compose input | Default | Purpose |
| --- | --- | --- |
| SHARED_SECRET | Required, ≥32 UTF-8 bytes | Authentication key shared by relay and robots |
| BIND_ADDRESS | 127.0.0.1 | Host address publishing relay |
| STRONG_NODE_PUBLISHED_PORT | 65432 | Host relay port; 0 selects an ephemeral port in tests |
| SOCKET_TIMEOUT | 5 | Socket connection/read timeout in seconds |
| DASHBOARD_PUBLISHED_PORT | 5173 | Host dashboard HTTP and /ws port; 0 for ephemeral test port |
| EXECUTOR_STEP_DELAY | 0.3 | Simulation navigation delay per step; 0 for fast tests |

Compose configures `NETWORK_HOST=strong-node`, `NETWORK_PORT=65432` for robots;
`COMMAND_POST_HOST=command-post`, `COMMAND_POST_PORT=65433` for the relay;
`COMMAND_POST_BIND=0.0.0.0`, `STRONG_NODE_HOST=0.0.0.0`,
`STRONG_NODE_PORT=65432`, and `STORAGE_FILE=/data/command_post_beacons.json`.
These variables are available for direct scripts too; robot host defaults to
127.0.0.1 outside Docker, replacing the old hardcoded LAN address. Use a positive
SOCKET_TIMEOUT and nonnegative EXECUTOR_STEP_DELAY.

`docker/.env.example` lists optional settings; append those you want to override
to the generated file. Restart/recreate affected containers with `docker compose
up -d` after changes; `restart` alone retains their existing environment.

Read [security.md](security.md) before publishing the relay remotely.

## Webapp integration

See [webapp.md](webapp.md) for internal MQTT/HTTP addresses and same-origin WebSocket
configuration. Compose passes no SHARED_SECRET to the browser or backend gateway;
only the Python protocol peers and network bridge receive it. Runtime frontend
settings are not Vite build settings. The default image derives /ws from its
browser origin. Optional ROS-demo ports are controlled by ROS_DASHBOARD_PORT (5174)
and FOXGLOVE_PUBLISHED_PORT (8765) in the standalone Compose file.

## Full ROS deployment settings

| Input | Default | Purpose |
| --- | --- | --- |
| COMPOSE_FILE | compose.yaml:compose.ros.yaml | Full mode; explicit -f selects alternatives |
| FOXGLOVE_PUBLISHED_PORT | 8765 | Localhost Foxglove WebSocket |
| ROS_BEACON_INTERVAL | 30 | Seconds between stored simulated writer beacons |

The override sets `EXECUTOR_MODE=ros`, `SIM_ONA=0`, `NETWORK_BEACONS=1`,
`MQTT_HOST=mosquitto`, `NETWORK_HOST=strong-node`. ROS receives the same private
HMAC key; no secret is sent to the frontend. Workspace `.dockerignore` excludes
secret files in the ROS build context; explicit COPY selects only source files.

## Migrated simulator settings

`AUTO_DISPATCH` defaults to `1` in both ROS deployments. Set it to `0` for manual
assignment. `SIM_SEED` is an optional integer. `SIM_COMMANDS=1` is set only on the demo backend;
it enables zone editing. Reset map is available in the integrated deployment too,
clearing the current session while preserving stored beacon records.
`shared/zone.json` is the canonical polygon config; ROS images set `ZONE_FILE` to the
copied config. Dashboard ellipse controls are build-time Vite values. See
[simulation documentation](../web-app/docs/simulation.md) for semantics and limitations.
