# Security and runtime limitations

All processes run as UID/GID 10001, with read-only root filesystems, dropped
capabilities, no-new-privileges, and process/CPU/memory limits. Only the command
post volume and ephemeral `/tmp` are writable. Health checks read the mission
through the real protocol rather than only testing for a process. Secrets are
absent from images and logs. Logs do include mission coordinates; restrict access.

HMAC authenticates but does not encrypt TCP traffic. The internal command post
is unauthenticated and intended for trusted peers. Keep it unpublished and allow
only trusted containers onto the Compose network. The host relay is localhost-only
unless BIND_ADDRESS is deliberately changed. There is no HTTP API/CORS/debug mode.
No Python dependency CVEs are added; base image CVE scanning still requires a
scanner. Docker Scout was previously blocked by Docker login; no clean scan is
claimed.

Existing application limits remain: sequential request handling, timestamp-only
replay keys reset on relay restart, one accepted beacon per second, entire JSON
missions held in memory, nearest-target selection regardless of age/type, and no
mission assignment/acknowledgement state. The network validates and translates
beacons; it does not implement an AI analysis pipeline. The executor simulates
navigation and reading a beacon; it does not physically rescue a victim or control
robot hardware. See [the refactor proposal](code-structure.md).

## Secret lifecycle

The same SHARED_SECRET is required for all protocol peers. Use random keys, keep
`docker/.env` private, and never paste the key into logs or documentation. Rotate
the key by updating all peers together and recreating the containers. Existing
external clients using the old embedded key must be reconfigured.

## Data and deployment boundaries

Beacon coordinates and events may be sensitive; control volume, backup and log
access. The command post trusts its network peers and has no independent
authentication. The current setup is a trusted-network simulation deployment,
with a localhost host binding by default. An internet deployment requires
transport protection, access control and a review of the current mission/replay
semantics. No clean vulnerability scan has been completed.
