# Run the complete system

## Requirements

Install Docker Engine with Docker Compose, or Docker Desktop with Compose. No
host Python, pip, Node.js, database, Redis or GPU runtime is required.
The webapp uses Node inside Docker. The optional
`docker/test.sh` helper uses a POSIX shell; Windows users can run it through WSL.
Use a recent Compose release supporting profiles, health dependencies and `wait`.
The executed verification used Compose v5.5.1.

Confirm Docker is running:

```bash
docker version
docker compose version
```

## First setup

Open a terminal in the workspace root (`aess/`), which contains `Network/`,
`Robots-Network/`, `docker/` and `docs/`:

```bash
cd docker
```

The active secret file is `docker/.env`. If it already exists, keep it. For a
**fresh checkout only**, generate it with container Python:

```bash
docker run --rm python:3.13-slim-bookworm@sha256:5024f48ba9441d4b13a95d3945abc6365538e3a31109833367a1923523c6efed python -c 'import secrets; print("SHARED_SECRET=" + secrets.token_hex(32))' > .env
printf '%s\n' 'COMPOSE_FILE=compose.yaml:compose.ros.yaml' >> .env
chmod 600 .env
```

This creates the required random key without installing host Python. Optional
settings and descriptions are in `.env.example` and [configuration.md](configuration.md).
Do not commit `.env`. The key is passed to all three authenticated peers by Compose.

Validate the configuration without printing resolved secrets:

```bash
docker compose -f compose.yaml --profile mission config --quiet
```

## Start the dashboard and assign a mission

From `aess/docker/`:

```bash
docker compose up -d --build --wait --wait-timeout 180
docker compose ps
```

Open **http://localhost:5173**, select a fresh writer beacon and assign it to the
executor. The ROS writer scans and creates beacons automatically. Commands use
the shared MQTT broker and ROS executor; progress and completion return through
the dashboard WebSocket. Connect Foxglove to **ws://localhost:8765** for live
sensors. See [ros-integration.md](ros-integration.md) for the full flow.

## Preserved Python CLI demonstration

This is an alternative to full ROS mode. Stop ROS and its bridge before switching:

```bash
docker compose stop ros network-bridge
```

Use explicit `-f compose.yaml` in every command in the CLI sections below.

Run from `aess/docker/`:

```bash
docker compose -f compose.yaml --profile mission up -d --build
docker compose -f compose.yaml wait writer executor
docker compose -f compose.yaml --profile mission ps -a
docker compose -f compose.yaml logs writer executor
```

Expected sequence:

1. Command post starts and passes its mission-retrieval health check.
2. Strong node starts and passes an authenticated health request through the command post.
3. Writer simulates a scan, drops a VICTIM beacon at (10.5, 0), sends it and receives a storage acknowledgement.
4. After the writer exits successfully, executor requests the stored mission through the strong node.
5. Executor selects the nearest beacon, navigates to it and reads nearby beacons.

Expected status:

| Service | Successful status |
| --- | --- |
| command-post | Running, healthy |
| strong-node | Running, healthy |
| mosquitto, backend, network-bridge, executor-control, dashboard | Running, healthy |
| writer | Exited (0) |
| executor | Exited (0) |

The writer/executor are finite jobs. `Exited (0)` means success. `up -d` returning
success only confirms startup; inspect the jobs after `wait` to verify completion.
Executor logs contain `Mission completed:` followed by JSON with target ID,
arrival coordinates and readings. A failed writer prevents executor startup.
Use `--wait` when starting only the long-running servers, not the finite jobs.

The public host address is `127.0.0.1:65432`, using a custom binary TCP protocol.
The dashboard is at http://localhost:5173; its `/ws` proxy reaches the private
backend. The TCP port has no HTTP API. Robots inside Docker use `strong-node:65432`.
The command post is private and its storage is retained in a named volume.

## Run another Python writer/executor cycle

From `aess/docker/`, retain the servers and stored records:

```bash
docker compose -f compose.yaml --profile mission rm -f writer executor
docker compose -f compose.yaml --profile mission up -d writer executor
docker compose -f compose.yaml wait writer executor
docker compose -f compose.yaml --profile mission ps -a
docker compose -f compose.yaml logs writer executor
```

Wait at least a second between writer runs because replay IDs use timestamp
seconds. Each run adds a stored beacon. Target selection considers every stored
beacon, so an older record can be chosen. The [architecture guide](architecture.md)
and [refactor proposal](code-structure.md) explain the current mission semantics.

To return to the full ROS deployment:

```bash
docker compose -f compose.yaml stop executor-control network-bridge
docker compose up -d --build --wait --wait-timeout 180
```

## Test the full flow in isolation

From the **workspace root**, not `docker/`:

```bash
./docker/test.sh
./docker/test-ros.sh
```

From `aess/docker/`, use `./test.sh` and `./test-ros.sh`. The base helper builds the images,
runs Python, backend, frontend and ROS bridge tests against an empty temporary mission store, recreates the servers to
check persistence, then cleans up only its temporary project. Normal mission
records stay intact. See [testing.md](testing.md) for individual test commands.

## Logs, restart and stop

From `aess/docker/`:

```bash
docker compose logs -f
docker compose restart ros backend network-bridge
docker compose --profile mission down
```

`down` removes containers but retains the normal mission volume. Do not add `-v`
unless deliberately deleting that deployment's data. Changing environment
settings requires `up -d` to recreate affected containers; `restart` retains their
old settings. See [operations.md](operations.md) for backup and restore.

If anything fails, use [troubleshooting.md](troubleshooting.md). This application
simulates robots, displays beacons and accepts dashboard assignments. AI analysis,
physical robot control and durable mission completion state are not implemented.
