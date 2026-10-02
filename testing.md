# Testing and verification

From the workspace root:

```bash
./docker/test.sh
```

The script uses a unique `aess-test-*` Compose project, ephemeral host port and
fresh volume; it does not clear or change the normal mission store. It builds
both images, starts healthy servers, runs 8 robot tests (including the two real
CLI processes and a bad-secret check), then 6 network tests. It recreates both
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
Physical robot motion, separate NGO dispatch, AI analysis and production load are
not covered because those implementations do not exist in this source.
