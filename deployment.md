# Docker deployment reference

All Dockerfiles, Compose files and environment examples are in `docker/`. Each
component keeps its `.dockerignore` at the source build context. Instructions
assume the current flat Python modules; see [development.md](development.md)
before implementing the proposed src-package migration.

| File | Purpose |
| --- | --- |
| docker/network.Dockerfile | Network server/test image built from Network/ |
| docker/robots.Dockerfile | Writer/executor/test image built from Robots-Network/ |
| docker/compose.yaml | Complete runtime, mission profile and test profile |
| docker/compose.dev.yaml | Read-only source mounts for fast iteration |
| docker/.env.example | Documented configurable inputs |
| docker/test.sh | Build, test and persistence verification in an isolated project |

Both images use the official Python 3.13 slim Bookworm image pinned to the digest
recorded in their Dockerfiles. They install no additional system or Python
packages. Python bytecode generation is disabled and stdout/stderr are unbuffered.
Sources and tests are explicitly copied; secret files and existing mission data
are excluded. Processes run as UID/GID 10001. Commands use the exec form and
Compose init forwards signals/reaps processes.

| Service | Image | Command | Profile |
| --- | --- | --- | --- |
| command-post | aess-network:local | python command_post_server.py | Default |
| strong-node | aess-network:local | python strong_node_server.py | Default |
| writer | aess-robots:local | python writer_client.py | mission |
| executor | aess-robots:local | python executor_client.py | mission |
| robot-tests | aess-robots:local | unittest discover in tests/ | test |
| network-tests | aess-network:local | unittest discover in tests/ | test |

The normal project name is `aess-network`. The command post and strong node have
`unless-stopped` restart policies. Mission/test jobs have no automatic restart.
Strong node waits for command-post health; writer waits for strong-node health;
executor waits for both strong-node health and successful writer completion.

Common constraints: read-only root filesystem, all capabilities dropped,
no-new-privileges, 64-process limit, 256 MB memory limit and 1 CPU limit per
container. A 16 MB ephemeral tmpfs is mounted at `/tmp`. Command post also has the
persistent beacons volume at `/data`. Logs rotate across three 10 MB JSON files.
Resource limits are starting values for the current small simulation; assess
memory and throughput before growing mission storage or adding workloads.

Health checks run every 15 seconds, with a 10-second Docker timeout, 3 retries
and 5-second start period. Command post returns its real mission list; strong node
uses an authenticated request through the command post. Checks do not add records.

Development mounts source read-only over `/app`; restart scripts or rerun finite
jobs after editing. Production-style base Compose uses baked sources and has no
debug or reload process. Configuration migration keeps the same project/volume
name, preserving existing deployment data.

Use the [run guide](run-guide.md), [configuration reference](configuration.md)
and [operations guide](operations.md) for executable commands. Upstream details:
[Compose profiles](https://docs.docker.com/compose/how-tos/profiles/) and
[startup dependencies](https://docs.docker.com/compose/how-tos/startup-order/).
