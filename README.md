# AESS documentation

Start with the [run guide](run-guide.md) to launch the complete Docker system.
The current code is a beacon-network and robot simulation using Python's standard
library. Docker and Docker Compose run all Python processes on the host's behalf.

| Guide | Contents |
| --- | --- |
| [Run guide](run-guide.md) | Requirements, first setup, launch, expected results, repeat a mission, stop |
| [Architecture](architecture.md) | Current components, actual flow, module responsibilities, missing future features |
| [Protocol](protocol.md) | Binary message layout, HMAC, responses, validation and JSON record schema |
| [Configuration](configuration.md) | Environment inputs, internal addresses, secrets and settings changes |
| [Docker deployment](deployment.md) | Images, build contexts, profiles, resource limits and writable paths |
| [Testing](testing.md) | Unit and full-system commands, isolation, assertions and limitations |
| [Development](development.md) | Source mounts, iteration, adding services and repository ownership |
| [Operations](operations.md) | Status, logs, persistence, backup, restore and original-data import |
| [Troubleshooting](troubleshooting.md) | Startup, authentication, replay, storage and mission-selection issues |
| [Security](security.md) | Secret handling, trusted network boundaries and runtime limitations |
| [Code structure proposal](code-structure.md) | Recommended folders, exact module mappings and staged migration |
| [Verification report](verification.md) | Previously executed build, tests and end-to-end results |

## Workspace organization

```text
aess/
├── README.md            # Short project entry point
├── docs/                # Full documentation lives here
├── docker/              # Dockerfiles, Compose, environment example, test helper
├── Network/             # Existing network component repository
└── Robots-Network/      # Existing robot component repository
```

This describes the current layout. The proposed package organization in
[code-structure.md](code-structure.md) is a recommendation, not an already-applied
source refactor. Runtime commands in these guides target the actual current code.
