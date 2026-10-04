# Security and runtime limitations

Processes use non-root image identities (Python 10001, Node 1000, nginx and Mosquitto), with read-only root filesystems, dropped
capabilities, no-new-privileges, and process/CPU/memory limits. Only the command
post volume and ephemeral `/tmp` are writable. Health checks read the mission
through the real protocol rather than only testing for a process. Secrets are
absent from images and logs. Logs do include mission coordinates; restrict access.

HMAC authenticates but does not encrypt TCP traffic. The internal command post
is unauthenticated and intended for trusted peers. Keep it unpublished and allow
only trusted containers onto the Compose network. The host relay is localhost-only
unless BIND_ADDRESS is deliberately changed. The dashboard publishes HTTP and its same-origin /ws proxy; private HTTP control
endpoints are unpublished. There is no debug/reload mode in the default images.
No Python dependency CVEs are added; base image CVE scanning still requires a
scanner. Docker Scout was previously blocked by Docker login; no clean scan is
claimed.

Existing application limits remain: sequential request handling, timestamp-only
replay keys reset on relay restart, one accepted beacon per second, entire JSON
missions held in memory, nearest-target selection regardless of age/type, and no durable
mission assignment state. The new dashboard adapter has in-memory mission state. The network validates and translates
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

The Node backend and dashboard production dependencies were audited with npm
audit --omit=dev during integration and reported zero vulnerabilities. This does
not certify the base OS images, dev dependencies or optional ROS environment.
