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
