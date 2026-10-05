# Testing and verification

From the workspace root:

```bash
./docker/test.sh
./docker/test-ros.sh
```

The script uses a unique `aess-test-*` Compose project, ephemeral host port and
fresh volume; it does not clear or change the normal mission store. It builds
both images, starts healthy servers, runs 11 robot tests (including the two real
CLI processes, a bad-secret check and mission-control tests), then 6 network tests, 26 backend/integration tests, 15 dashboard tests and the
existing ROS bridge/model tests. It recreates both
servers and checks that both test beacons survived. It prints status/logs and
removes only its temporary project and volume, including on failure. Any test
failure returns a nonzero exit code.

From `aess/docker/`, run individual unit tests against the normal stack without adding beacons:

```bash
docker compose run --rm -T robot-tests python -m unittest discover -s tests -p test_robots.py -v
docker compose run --rm -T network-tests python -m unittest discover -s tests -k UnitTests -v
```

Full robot tests intentionally require an empty mission store. Do not run them
against a populated normal volume. The network integration test appends a beacon.
No SQL database or migration command is applicable.

## What the tests prove

| Suite | Coverage | Expected count |
| --- | --- | --- |
| Robots unit | scan/drop, attrition, beacon codec, PoD decay, radio range, acknowledgement validation, mission validation, navigation/read | 6 |
| System | actual writer and executor CLI processes against live network containers; wrong-secret job rejection | 2 |
| Network | codec/GPS, HMAC, persistence/corruption, failed writes, retry after failed forwarding, authenticated live protocol and error handling | 6 |

The end-to-end test verifies that the executor arrives at and reads the exact
network-assigned beacon created by the writer. It starts with an empty store and
does not delete application data itself. `docker/test.sh` removes its own unique
project and volume after completion. The persistence check verifies both test
records survive forced server recreation.

A passing build alone is insufficient. Inspect service health and both finite
job exit codes. Historical executed results are in [verification.md](verification.md).
Physical robot motion, AI analysis and production load are
not covered because those implementations do not exist in this source.

## Webapp tests

The backend suite requires a real broker in docker/test.sh, so its MQTT test does
not skip. The integrated test exercises dashboard HTML/assets, nginx /ws upgrades,
MQTT forwarding, selected-beacon Python navigation/read and cancellation. It is
not a browser rendering test. ROS bridge tests run without requiring a full ROS
image. The separate full ROS integration is exercised by `test-ros.sh`.

## Full ROS integration

See [ros-integration.md](ros-integration.md) for the current complete deployment,
Foxglove connection, executor ownership, health checks and mode switching. Run
`./docker/test-ros.sh` from the workspace root for live ROS/Foxglove/mission checks.

## Migrated simulation workflow

Run `./docker/test-simulation.sh` from the workspace root. It verifies automatic mission
completion, reset notifications to two live dashboards, live zone editing and the zone
in a late client's sync snapshot. The helper creates and cleans an isolated project.
