# AESS integrated robot network

ROS Writer scans and sends beacons through the authenticated Python network to
persistent storage. The dashboard displays them, assigns missions to the ROS
Executor, and receives progress and completion. Foxglove streams live ROS sensors.
Robots are simulated; physical control and AI reasoning are not implemented.

Full documentation lives in [docs/](docs/README.md). Start with the
[run guide](docs/run-guide.md) and [ROS integration guide](docs/ros-integration.md).

With the existing `docker/.env`:

```bash
cd docker
docker compose up -d --build --wait --wait-timeout 180
docker compose ps
```

Open **http://localhost:5173**. The ROS Executor automatically visits Writer beacons;
set `AUTO_DISPATCH=0` for manual beacon selection and mission assignment.
Connect Foxglove to **ws://localhost:8765**. The writer creates beacons automatically.
Seven services run in full ROS mode; the alternative Python HTTP executor is disabled.

From the workspace root:

```bash
./docker/test.sh       # Existing component and Python integration tests
./docker/test-ros.sh   # Full ROS/Foxglove/dashboard integration tests
```

For first-time setup, stop/restart, logs and data management, follow the run guide.
`Network/`, `Robots-Network/` and `web-app/` are separate Git repositories; shared
`docker/` and `docs/` belong in a parent integration repository.
