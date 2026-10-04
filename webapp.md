# Webapp integration

The main deployment now runs the **full ROS system**. Start with
[ros-integration.md](ros-integration.md): the ROS Writer generates beacons, the
existing Python network persists them, dashboard commands drive the ROS Executor,
and Foxglove streams live sensors. Full runtime verification passed.

The sections below describe the preserved **Python-only alternative**. Select it
explicitly with `docker compose -f compose.yaml` after stopping ROS/its bridge as
documented in that guide. Do not run both executor command consumers together.

## Start and use

Only Docker and Compose are required on the host. Keep the existing private
`docker/.env`; fresh checkouts use the secret generation command in the
[run guide](run-guide.md). Run from `aess/docker/`:

```bash
docker compose -f compose.yaml up -d --build --wait
docker compose -f compose.yaml ps
# Create a beacon using the real existing Python writer workflow:
docker compose -f compose.yaml run --rm -T writer
```

Open **http://localhost:5173**. The dashboard should show an open WebSocket link
and a connected ONA/MQTT backend. Stored Python beacons appear in Beacons and as
map targets. Select a beacon and click the executor assignment button. Observe
mission status and executor position. Cancel an active mission to stop navigation.
Old beacons preserve their original source timestamp and may appear STALE/LOST;
they are not rewritten as fresh observations by the bridge.

The optional `mission` profile still supports the earlier writer + automatic
nearest-beacon executor CLI demo. Dashboard assignment uses `executor-control`
and the exact selected beacon ID; it does not use that one-shot executor. The
writer CLI is a finite scan, so continuous writer odometry/battery streams are not
fabricated. Physical robot drivers, SLAM and AI analysis remain outside this stack.

## Actual architecture

```text
Writer → authenticated strong-node → command-post → beacon JSON volume
                              ↑ authenticated mission-list polling
                       network-bridge → MQTT → backend → dashboard /ws
                              ↑                        ↑
                   MQTT cmd/assign-mission ← backend ← dashboard command
                              ↓
                      executor-control → existing Python Executor
                              ↓ navigation/read/status
                       network-bridge → MQTT → backend → dashboard
```

| Service | Purpose | Host access |
| --- | --- | --- |
| command-post | Existing persistent beacon store | Private |
| strong-node | Existing authenticated binary TCP relay | 127.0.0.1:65432 |
| mosquitto | Persistent retained MQTT observations | Private |
| backend | Existing contract normalization, WebSocket snapshots and commands | Private |
| network-bridge | Binary-network → MQTT conversion; command forwarding and executor status | Private |
| executor-control | Private standard-library Python HTTP adapter using existing Executor/WeakNode logic | Private |
| dashboard | Built React assets served by non-root nginx; /ws reverse proxy | 127.0.0.1:5173 |

The browser uses its own origin for `/ws`, including `wss` on HTTPS. Container DNS
addresses remain internal. The dashboard no longer bakes in a localhost backend
address or needs VITE_WS_URL passed as runtime environment. Non-Docker development
can set VITE_WS_URL explicitly at Vite build/dev time.

Vite preview was replaced with a static runtime because [Vite documents preview
as a local preview server](https://vite.dev/guide/static-deploy), not a production
server. Dockerfiles use pinned official Node 22 Alpine and nginx images and locked
`npm ci` dependencies. Python continues to use its pinned official 3.13 image.

## Adapter behavior

`web-app/backend/src/network-bridge-core.js` reads the strong node's existing
0x02 + HMAC mission request and validates stored records. Conversion preserves
node ID, coordinates, source Unix timestamp and PoD. It publishes retained
`beacons/<id>` and `targets/<id>` using the existing shared JS contract.

`network-bridge.js` subscribes to non-retained `cmd/#` commands. It ignores
retained commands to prevent replay on restart, validates command envelopes,
and forwards assignments/cancellations to `executor-control`. It publishes
executor telemetry and mission transitions once per second. Failed commands emit
system warning events. The backend's `cmd.ack: received` acknowledges transport
receipt, not mission completion; subsequent mission messages report execution.

`Robots-Network/mission_service.py` checks the selected beacon against the actual
authenticated network snapshot, starts navigation using the existing Executor,
and records real beacon readings on arrival. One mission runs at a time;
concurrent assignments are rejected. Repeated current mission IDs return the same
mission state. Cancellation is checked between navigation steps. The original
CLI behavior is preserved by an optional cancellation callback in navigate_to.

The adapter's private HTTP API is GET `/health`, GET `/state`, POST `/missions`
with id/beaconId/objective, and POST `/cancel` with missionId. It is deliberately
unpublished and trusts the integration network. It is not an external robot API.

## Environment, health and persistence

`DASHBOARD_PUBLISHED_PORT` defaults to 5173 in `docker/.env.example`. Existing
BIND_ADDRESS, SHARED_SECRET and EXECUTOR_STEP_DELAY settings apply. Internal
settings are fixed explicitly in Compose: MQTT_URL=mqtt://mosquitto:1883,
EXECUTOR_URL=http://executor-control:8080, NETWORK_HOST=strong-node and
NETWORK_PORT=65432. Backend listens on 4311 internally.

Backend health requires an MQTT connection. Bridge health requires recent
successful polling of both the authenticated network and executor service.
Broker health reads its version topic. Dashboard health checks nginx availability.
All seven default services use health checks and dependency readiness.

The original beacon volume is preserved. `aess-network_mqtt-data` persists broker
retained data. Mission-control state and backend snapshots are in memory; restart
can lose active assignments, and retained mission messages can remain historical.
This is a simulation integration, not durable mission scheduling. Broker restore
and retention cleanup require care if beacon backups are rolled back: retained
older beacon topics are not automatically deleted.

## Development and tests

From `aess/docker/`:

```bash
docker compose -f compose.yaml -f compose.dev.yaml up -d --build --wait
# Python service / Node source changes:
docker compose -f compose.yaml -f compose.dev.yaml restart executor-control backend network-bridge
# Dashboard asset/config changes require rebuilding the static image:
docker compose up -d --build dashboard
```

From the workspace root, `./docker/test.sh` runs the existing Python suites, new
mission-control tests, backend tests (including real MQTT and dashboard-WebSocket
assignment/cancellation), frontend Vitest tests and the pure-Python ROS bridge
suite. It uses an isolated volume/project, then verifies byte-for-byte beacon
persistence after recreating the stack. Vite test-only temporary mounts are
writable; the production dashboard remains read-only.

The integrated test fetches HTML/assets and exercises the same `/ws` route as the
browser. It does not perform browser rendering or visual interaction assertions.

## Optional standalone ROS/Foxglove demo

The original ROS writer/executor/ONA simulators are preserved separately. Do not
run them on the default integration broker: two producers for the same robot IDs
would mix different simulations. To explicitly select the standalone demo:

```bash
cd docker
docker compose -f compose.ros-demo.yaml up -d --build
# Dashboard: http://localhost:5174; Foxglove: ws://localhost:8765
docker compose -f compose.ros-demo.yaml down
```

This uses a distinct project and broker, defaults to dashboard port 5174, and may
need a large ROS image build. The ROS image uses Humble and paho-mqtt 2.1.0.
Its pure bridge/model tests run in the default test workflow. The integrated ROS
image and complete stack have now been built and runtime-tested; this distinct
standalone fake-ONA demo was not separately exercised.
See the [webapp contract](webapp-contract.md) for its topic shapes and source
[ROS tools](../web-app/ros2/tools/) for development utilities.

## Security and remaining limits

Default integration processes run non-root with read-only roots, dropped
capabilities and bounded resources. Secrets are excluded from build contexts.
MQTT, backend and executor-control are private. Localhost dashboard/TCP publishing
is the default. MQTT and operator commands remain unauthenticated within this
trusted local deployment; add authentication and TLS before remote exposure.
HMAC authenticates the Python binary protocol but does not encrypt it.

Timestamp-only beacon replay IDs, sequential TCP server handling, JSON storage
and simulated navigation remain unchanged. Dashboard targets follow shared JS
staleness/PoD thresholds, which differ from the WeakNode decay model; those are
separate existing policies and require a deliberate domain decision to unify.
