# Operations and persistent storage

Run commands from `aess/docker/`.

## Status, restart and stop

```bash
docker compose --profile mission ps -a
docker compose logs -f
docker compose restart command-post strong-node
docker compose --profile mission down       # retain normal mission data
```

Use the mission profile when stopping to include completed job containers.
`docker compose down -v` deletes the selected project's volume; do not use it
for the normal project unless intentionally deleting mission data.

## Storage and backup

The Compose project stays named `aess-network` so moving configs does not rename
or replace the existing `aess-network_beacons` volume. Its JSON mission file is
mounted at `/data/command_post_beacons.json`. The original tracked JSON file in
`Network/` is untouched and excluded from the image. Robots have no persistent
hardware state; simulation completion is reported to stdout as JSON, with target
ID, arrival coordinates and beacon readings. Docker logs rotate at 3 × 10 MB per
container. `/tmp` is a bounded 16 MB ephemeral tmpfs for tests and temporary files.

Backup with servers stopped:

```bash
docker compose stop command-post strong-node
docker compose run --rm --no-deps -T command-post python -c 'from pathlib import Path; print(Path("/data/command_post_beacons.json").read_text())' > beacons-backup.json
docker compose up -d --wait
```

For an optional import into a **new, empty volume**, before starting the servers:

```bash
docker compose run --rm --no-deps -T command-post python -c 'import json, pathlib, sys; p=pathlib.Path("/data/command_post_beacons.json"); data=json.load(sys.stdin); assert isinstance(data,list); assert not p.exists(), "Volume already has data; import refused"; p.write_text(json.dumps(data))' < ../Network/command_post_beacons.json
```

No import is needed for the existing workspace. Never import over a running server.

## Restore a backup

This procedure replaces the selected deployment's mission file. Check that the
backup belongs to that deployment. Stop both servers first; retain the volume.

```bash
docker compose stop command-post strong-node
docker compose run --rm --no-deps -T command-post python -c 'import json, sys; import command_post_server as cp; data=json.load(sys.stdin); assert isinstance(data,list), "Backup must be a list"; cp.stored_beacons=data; cp.save_beacons(); print("Restored", len(data), "beacons")' < beacons-backup.json
docker compose up -d --wait
```

Restoration uses the application's atomic write function and the configured
STORAGE_FILE. Keep backups outside the Git repository; they contain mission data.
There are no SQL migrations.

## Logs and completion

Use `docker compose logs writer executor` for finite jobs and
`docker compose logs -f command-post strong-node` for running servers. A successful
executor emits a JSON completion result with target ID, arrival coordinates and
readings. This result is a simulation log, not durable assignment state. Logs
rotate; archive results separately if they must be retained long term.
