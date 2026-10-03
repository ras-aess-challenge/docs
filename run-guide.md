# Run the complete system

## Requirements

Install Docker Engine with Docker Compose, or Docker Desktop with Compose. No
host Python, pip, Node.js, database, Redis or GPU runtime is required. The optional
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
chmod 600 .env
```

This creates the required random key without installing host Python. Optional
settings and descriptions are in `.env.example` and [configuration.md](configuration.md).
Do not commit `.env`. The key is passed to all three authenticated peers by Compose.

Validate the configuration without printing resolved secrets:

```bash
docker compose --profile mission config --quiet
```

## Launch the complete demonstration

Run from `aess/docker/`:

```bash
docker compose --profile mission up -d --build
docker compose wait writer executor
docker compose --profile mission ps -a
docker compose logs writer executor
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
| writer | Exited (0) |
| executor | Exited (0) |

The writer/executor are finite jobs. `Exited (0)` means success. `up -d` returning
success only confirms startup; inspect the jobs after `wait` to verify completion.
Executor logs contain `Mission completed:` followed by JSON with target ID,
arrival coordinates and readings. A failed writer prevents executor startup.
Use `--wait` when starting only the long-running servers, not the finite jobs.

The public host address is `127.0.0.1:65432`, using a custom binary TCP protocol.
There is no browser page or HTTP API. Robots inside Docker use `strong-node:65432`.
The command post is private and its storage is retained in a named volume.

## Run another writer/executor cycle

From `aess/docker/`, retain the servers and stored records:

```bash
docker compose --profile mission rm -f writer executor
docker compose --profile mission up -d writer executor
docker compose wait writer executor
docker compose --profile mission ps -a
docker compose logs writer executor
```

Wait at least a second between writer runs because replay IDs use timestamp
seconds. Each run adds a stored beacon. Target selection considers every stored
beacon, so an older record can be chosen. The [architecture guide](architecture.md)
and [refactor proposal](code-structure.md) explain the current mission semantics.

To run only the servers:

```bash
docker compose up -d --build --wait
```

## Test the full flow in isolation

From the **workspace root**, not `docker/`:

```bash
./docker/test.sh
```

From `aess/docker/`, the equivalent is `./test.sh`. The helper builds both images,
runs 14 tests against an empty temporary mission store, recreates the servers to
check persistence, then cleans up only its temporary project. Normal mission
records stay intact. See [testing.md](testing.md) for individual test commands.

## Logs, restart and stop

From `aess/docker/`:

```bash
docker compose logs -f
docker compose restart command-post strong-node
docker compose --profile mission down
```

`down` removes containers but retains the normal mission volume. Do not add `-v`
unless deliberately deleting that deployment's data. Changing environment
settings requires `up -d` to recreate affected containers; `restart` retains their
old settings. See [operations.md](operations.md) for backup and restore.

If anything fails, use [troubleshooting.md](troubleshooting.md). This application
simulates robots and retrieves mission lists; separate NGO dispatch, AI analysis,
physical robot control and durable mission completion state are not implemented.
