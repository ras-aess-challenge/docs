# Full ROS integration

## What runs

The main deployment combines the existing authenticated Python network, React
webapp and ROS 2 Humble simulation. ROS uses Ubuntu's Python 3.10; the original
network/CLI containers use Python 3.13. No GPU or host Python/Node/ROS is needed.

```text
ROS Writer: odometry + lidar simulation
  → network_writer_node (original Writer + authenticated writer_client)
  → strong-node → command-post → persistent beacon JSON
  → network-bridge (authenticated polling) → shared MQTT broker
  → backend → dashboard
Dashboard assignment → backend → MQTT cmd/assign-mission
  → ros_mqtt_bridge → /executor/mission → ROS Executor
  → ROS status → MQTT missions/robots/events → backend → dashboard
Foxglove ← ROS sensors, robot health, network beacons and targets
```

There is one broker and one active executor command consumer. The full override
sets `EXECUTOR_MODE=ros` and disables the Python HTTP executor. `SIM_ONA=0`
disables the standalone fake ONA producer. The actual network validates, stores
and returns beacons; it does not implement an AI reasoning or NGO dispatch engine.
Assignments originate from the operator dashboard. Executor movement and inspection
are simulated, not physical navigation or a real victim verification sensor.

## Launch and use

From `aess/docker/`, the private `.env` selects full integration using
`COMPOSE_FILE=compose.yaml:compose.ros.yaml`. Fresh setup is in [run-guide.md](run-guide.md).
The explicit equivalent is:

```bash
docker compose -f compose.yaml -f compose.ros.yaml up -d --build --wait --wait-timeout 180
docker compose -f compose.yaml -f compose.ros.yaml ps
```

The ROS image is larger than the Python images and its initial build downloads
ROS/Foxglove dependencies. Seven long-running services should be healthy:
`command-post`, `strong-node`, `mosquitto`, `backend`, `network-bridge`, `ros`, `dashboard`.
A previously stopped `executor-control` container may still appear in `ps -a`.

1. Open **http://localhost:5173** and wait for live Writer/Executor telemetry.
2. Select a fresh writer beacon, then assign it to the Executor.
3. Observe pending → active → inspecting → done and updated position.
4. Assign another mission and cancel during navigation/inspection to test cancellation.
5. In Foxglove, choose a Foxglove WebSocket connection to **ws://localhost:8765**.
   Import `web-app/foxglove/living_map_robot_health.json` as a layout.

Foxglove exposes `/scan`, `/odom`, `/battery_state`, `/diagnostics`,
`/executor/status`, `/network/beacons`, `/network/targets` and other ROS channels.
Installed bridge 3.5.0 negotiates `foxglove.sdk.v1`; our probes and tests also
support `foxglove.websocket.v1`. Use a current Foxglove client. No host Foxglove
installation is required if using its browser client.

## Health, runtime and storage

The ROS service runs as UID/GID 10001, with a read-only root, writable `/tmp`,
dropped capabilities, init and supervised children. If any ROS child exits, the
container exits and its restart policy restarts the complete ROS group.
Readiness checks fresh odometry/executor telemetry, MQTT connectivity and an
actual Foxglove WebSocket handshake. Bridge readiness additionally checks live
executor MQTT telemetry and authenticated network polling.

`ROS_BEACON_INTERVAL` defaults to 30 seconds. Each drop is a persistent record;
there is currently no automated retention policy. Increase the interval or stop
the simulation when unused to limit store growth. Runtime ROS logs/health are
ephemeral under `/tmp`; Docker captures console logs. Existing named beacon and
MQTT volumes survive recreation. Mission state remains in memory; restarts do
not resume missions. Simulated lidar faults are intentional diagnostic exercises.

The official ROS base is pinned by digest and paho-mqtt by version. Apt ROS
packages come from the Humble repository at build time; exact package builds
are not snapshot-locked. Foxglove, tf2 and diagnostic messages are needed for
visualization/transforms/health; pip installs the MQTT client. No CUDA is installed.

## Verification

From the workspace root:

```bash
./docker/test-ros.sh
./docker/test.sh
```

The first helper uses an isolated project, random host ports and temporary
volumes. It builds ROS and the webapp, waits for every service to become healthy,
asserts live advertised ROS channels and actual binary lidar frames, and drives
assignment, inspection, completion and cancellation through the dashboard `/ws`
endpoint and actual network-generated beacon. It also runs the 19 pure ROS
bridge/model tests. It cleans up only its own project/volumes.
The second helper tests the preserved Python integration plus component suites.
Neither test is a physical robot test or a browser rendering test.

## Alternative modes

To switch an existing ROS deployment to the original Python HTTP executor,
stop the ROS/bridge producers first, then explicitly select the base file:

```bash
docker compose stop ros network-bridge
docker compose -f compose.yaml up -d --build --wait
```

To return to full ROS mode:

```bash
docker compose -f compose.yaml stop executor-control network-bridge
docker compose up -d --build --wait --wait-timeout 180
```

The preserved standalone `compose.ros-demo.yaml` uses its own broker/fake ONA;
it is not the full integrated system. It also uses port 8765, so stop integrated
ROS or choose another `FOXGLOVE_PUBLISHED_PORT` before launching that demo.
