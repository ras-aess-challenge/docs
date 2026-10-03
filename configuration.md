# Configuration

Run shell commands in this guide from `aess/docker/`. See the [run guide](run-guide.md) for first startup.

The active environment file is `docker/.env`; `Network/.env` is no longer used.
On a fresh checkout with no existing `.env`, generate a key as follows. If `.env`
already exists, keep it: these commands replace its contents and rotate the key.

```bash
# Generate the required secret using container Python:
docker run --rm python:3.13-slim-bookworm@sha256:5024f48ba9441d4b13a95d3945abc6365538e3a31109833367a1923523c6efed python -c 'import secrets; print("SHARED_SECRET=" + secrets.token_hex(32))' > .env
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
