# Troubleshooting

Run commands from `aess/docker/`. Start with:

```bash
docker compose --profile mission ps -a
docker compose logs --tail 100 command-post strong-node writer executor
```

- `Exited (0)` for writer/executor is successful completion, not a crashed server.
- A writer error means the executor must remain unstarted. Inspect `logs writer`.
- Authentication error: use the same secret on all clients; the old embedded key
  is no longer used.
- Unhealthy relay: check command-post logs, DNS and data volume permissions.
- Port occupied: change STRONG_NODE_PUBLISHED_PORT; test.sh uses an ephemeral port.
- No mission: run a writer first. Empty/error responses now fail the executor job.
- Replay/stale rejection: synchronize clocks; wait at least a second between
  scans. Network accepts ages up to 300 seconds and future skew up to 5 seconds.
- Corrupt JSON: restore a backup; startup fails instead of discarding existing data.
- Legacy JSON relay: do not start Robots-Network/strong_node_server.py with these
  clients. Network/strong_node_server.py handles the binary/HMAC protocol.
- New root config not tracked: the two existing repos do not contain `docker/`;
  version the integration directory in the parent integration repository.

## Tests fail with a nonempty store

Use `./docker/test.sh` from the workspace root. The full robot suite deliberately
requires an empty test store. Do not delete the normal volume to satisfy a test.

## Executor reads an older beacon

The current mission is the full stored list, and target selection uses nearest
distance only. Equal-distance records keep their existing order. This is
application behavior, not a Docker routing problem. The [code structure and
mission refactor proposal](code-structure.md) describes explicit assignments
and completion state.

## The writer ran but the executor did not start

Check writer logs and its exit code. Delivery, authentication and server errors
make the writer fail; Compose intentionally gates executor startup on successful
writer completion. Correct the cause, then follow the rerun instructions in the
[run guide](run-guide.md).

## Full ROS integration

See [ros-integration.md](ros-integration.md) for the current complete deployment,
Foxglove connection, executor ownership, health checks and mode switching. Run
`./docker/test-ros.sh` from the workspace root for live ROS/Foxglove/mission checks.

### Dashboard WebSocket returns 502 after backend recreation

The old nginx configuration resolved the backend only at startup. The current
config uses Docker DNS (`127.0.0.11`, 10-second cache) and a variable upstream,
and dashboard health proxies the backend's health endpoint. Rebuild dashboard
if running an older image; wait for DNS/cache/readiness after replacing backend.

### Foxglove responds 400 to a raw WebSocket check

Bridge 3.5.0 expects the `foxglove.sdk.v1` subprotocol. Use a current Foxglove
client or the provided test/probe offering SDK and legacy subprotocols. A plain
HTTP GET to port 8765 is not a valid Foxglove health check.
