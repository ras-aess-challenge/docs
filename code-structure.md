# Code organization and refactor plan

Keep Docker integration separate from protocol and mission redesign. The current
implementation preserves both projects and adds only a small shared robot
transport module and inspectable simulation results. Suggested next steps:

1. **Create importable packages with clear roles.** Move flat modules to
   `Network/src/aess_network/` and `Robots-Network/src/aess_robots/`; add
   `__init__.py`, `pyproject.toml` metadata and module entry points. Separate
   `domain/` (Writer, Executor, WeakNode), `transport/` (TCP client/server),
   `storage/` (JSON repository), and `cli/` (job entry points). Keep real domain
   behavior out of CLI startup and environment-loading code.
2. **Extract one versioned protocol package.** Both repos duplicate beacon codec,
   HMAC constants/helpers and GPS translation. A small shared `aess_protocol`
   package should own binary encoding, typed beacon records, event validation and
   authentication. First add cross-project wire-compatibility tests, then extract
   without changing bytes. Retire the unused JSON-only robot relay once callers
   have been audited. Avoid `sys.path` tricks and cross-repo imports.
3. **Make configuration explicit.** Load validated settings into dataclasses at
   startup and pass them into clients/servers instead of reading environment
   globals during import. This makes authentication, host names and timeouts
   easier to test without process-wide environment changes. Keep defaults and
   deployment examples in docker/ rather than copying settings between repos.
4. **Define mission state separately from beacon storage.** Currently a mission
   is every stored beacon and the executor picks the nearest, even if old. Add a
   typed Mission with ID, target beacon ID, action, status and assignment. Separate
   validation/GPS enrichment from future analysis and planning. Implement explicit
   dispatch/request and completion acknowledgements before adding a worker or
   scheduler. Agree whether an NGO sends assignments or executors poll; current
   code only implements polling.
5. **Version replay and delivery semantics.** Timestamp-only IDs reject multiple
   writers in the same second; replay state vanishes on restart. A versioned wire
   format with writer ID plus sequence/nonce allows independent writers, durable
   deduplication and idempotent delivery. This is a compatibility change requiring
   coordinated client/server rollout, not an incidental Docker fix.
6. **Decouple simulation from robot hardware.** Navigation, scanning and radio
   transmission should implement small interfaces. Keep today’s simulation as one
   implementation, inject a clock/sleeper for deterministic tests, and add hardware
   adapters only when devices/ROS or drivers exist. Keep event handling explicit;
   “mission complete” currently means arriving and reading a beacon.
7. **Keep storage modest until load requires more.** Hide JSON behind a repository
   interface and test persistence/error behavior. Add SQLite/PostgreSQL only when
   concurrent writers, mission assignment transactions or larger datasets require
   it. Use structured logs with event, node ID and request ID; avoid logging keys.

Suggested future layout:

```text
aess/
├── docs/                      # full documentation
├── docker/                    # build/deployment/integration settings
├── protocol/                  # versioned shared package (future)
├── Network/
│   ├── src/aess_network/
│   │   ├── transport/
│   │   ├── storage/
│   │   └── cli/
│   └── tests/
└── Robots-Network/
    ├── src/aess_robots/
    │   ├── domain/
    │   ├── transport/
    │   ├── simulation/
    │   └── cli/
    └── tests/
```

Deliver each step independently with the existing Docker end-to-end test passing.
Do not move all modules or change the wire protocol in one large rewrite. Both
component folders are separate Git repositories; version the shared integration
and future protocol package deliberately in a parent integration repository or
as versioned dependencies/submodules, rather than leaving shared files untracked.

## Concrete target structure

This is a proposed structure. Keep existing scripts as thin compatibility entry
points during migration; update Docker module commands only after container tests
pass. Keep the two component repository boundaries initially.

```text
aess/
├── docs/                              # Canonical documentation
├── docker/                            # Shared deployment/integration
├── protocol/                          # Future versioned shared package
│   ├── pyproject.toml
│   ├── src/aess_protocol/
│   │   ├── __init__.py
│   │   ├── beacon.py                  # Binary payload, events and Beacon type
│   │   └── authentication.py          # HMAC and message constants
│   └── tests/
├── Network/
│   ├── pyproject.toml
│   ├── src/aess_network/
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── services/
│   │   │   ├── beacon_service.py      # Validation and coordinate enrichment
│   │   │   └── mission_service.py     # Existing retrieval; later assignment
│   │   ├── transport/
│   │   │   ├── strong_node.py         # TCP listener and request handling
│   │   │   └── command_post.py        # TCP storage/retrieval endpoint
│   │   ├── storage/
│   │   │   └── json_repository.py     # Atomic storage, load/save
│   │   ├── geo/
│   │   │   └── coordinates.py
│   │   └── cli/
│   │       ├── strong_node.py
│   │       ├── command_post.py
│   │       └── healthcheck.py
│   └── tests/
│       ├── unit/
│       └── integration/
└── Robots-Network/
    ├── pyproject.toml
    ├── src/aess_robots/
    │   ├── __init__.py
    │   ├── settings.py
    │   ├── domain/
    │   │   ├── writer.py
    │   │   ├── executor.py
    │   │   └── weak_node.py
    │   ├── transport/
    │   │   └── network_client.py
    │   ├── simulation/
    │   │   └── radio.py
    │   └── cli/
    │       ├── writer.py
    │       └── executor.py
    └── tests/
        ├── unit/
        └── integration/
```

Do not add a module just to fill this tree. For a first packaging pass, keep the
network server logic together under transport/; extract services and storage in
later commits with focused tests. The shared protocol can be introduced after
both packages work independently.

## Module migration map

| Current file | Proposed destination | Work required |
| --- | --- | --- |
| Network/strong_node_server.py | aess_network/transport/strong_node.py + cli/strong_node.py | Move first; later separate validation from socket handling |
| Network/command_post_server.py | aess_network/transport/command_post.py + storage/json_repository.py + cli/command_post.py | Move listener, then extract storage without changing acknowledgements |
| Network/healthcheck.py | aess_network/cli/healthcheck.py | Use explicit settings and shared request helpers |
| Network/frame_translation.py | aess_network/geo/coordinates.py | Retain conversion and existing anchor defaults |
| Both beacon_codec.py | aess_protocol/beacon.py | Prove identical bytes and event behavior before deduplication |
| Both security_config.py | aess_protocol/authentication.py + each application's settings.py | Separate pure HMAC helpers from environment/secret loading |
| Robots-Network/writer.py | aess_robots/domain/writer.py | Preserve move/drop/attrition rules |
| Robots-Network/executor.py | aess_robots/domain/executor.py | Preserve target choice/navigation/read results |
| Robots-Network/weak_node.py | aess_robots/domain/weak_node.py | Preserve reconstruction and decay formula |
| Robots-Network/network_physics.py | aess_robots/simulation/radio.py | Inject randomness in tests; keep range/loss behavior |
| Robots-Network/network_client.py | aess_robots/transport/network_client.py | Pass settings/client dependencies explicitly |
| Robots-Network/writer_client.py | aess_robots/cli/writer.py | Thin startup around the existing workflow |
| Robots-Network/executor_client.py | aess_robots/cli/executor.py | Thin startup around the existing workflow |
| Robots-Network/frame_translation.py | Retire duplicate after checking consumers | Current client workflow does not use it |
| Robots-Network/strong_node_server.py | legacy/ until callers are audited, then retire | Do not deploy the incompatible JSON-only relay |
| Existing tests | Component tests/unit/ and tests/integration/ | Update imports, mock paths, discovery and subprocess module commands |

## Dependency direction and conventions

CLI loads settings and assembles collaborators. Transport handles wire IO and
calls application services. Services depend on domain objects and a repository
interface; storage implements that interface. Domain classes should not import
Docker settings, sockets or CLI startup. Both applications depend on the shared
protocol package, not on each other's source directories.

Use absolute package imports, descriptive module/function names, type annotations
for public interfaces, and short docstrings for protocol or nonobvious behavior.
Prefer small functions with explicit return values and exceptions over print-only
success signaling. Use logging at service boundaries; avoid scattering logging
and environment reads through domain classes. Keep tests beside their component,
runtime data in volumes, and bytecode/build outputs out of Git.

Future module commands could be `python -m aess_robots.cli.writer` and
`python -m aess_network.cli.strong_node`. These are **not current run commands**.
Package installation should happen inside Docker with explicit pyproject metadata;
do not make the host require pip or ad-hoc PYTHONPATH/sys.path modifications.

## Safe implementation order

1. Add packaging metadata and move one component at a time, preserving entry scripts.
2. Update imports, test mock targets, Docker COPY/install layers and development mounts.
3. Run unit tests and the full isolated Docker flow before moving the next component.
4. Add cross-component protocol tests and extract the shared protocol.
5. Extract settings/storage/service responsibilities behind explicit interfaces.
6. Add mission assignment/dispatch/completion semantics as a separate feature.

Acceptance checks: unchanged wire bytes and GPS/decay behavior, preserved volume
identity/data, non-root containers, real client round trips, writer-before-executor
ordering, and explicit failure on authentication/storage errors. Update these
docs and the run guide with every entry-point or configuration change.

## Webapp integration additions

The webapp already separates backend/, dashboard/, shared/ and ros2/. Keep that
component boundary. Move network-bridge-core.js and network-bridge.js into a
backend integrations/python-network/ package in a later organizational change.
mission_service.py belongs in the proposed aess_robots/transport and CLI layers;
keep selection/mission lifecycle in a service rather than the HTTP handler.
The adapter adds in-memory assignment/cancellation now; durable assignment remains
a separate refactor. Three component repositories now share docker/ and docs/.

## ROS integration boundaries

Keep `ros2/nodes/` for ROS subscriptions/publishers, `ros2/domain/` for pure
executor/mission/diagnostic logic, and `ros2/adapters/` for MQTT and authenticated
network transport. Move node launch supervision into an explicit launch package
only when these nodes are packaged; keep the current Docker entrypoint until then.
Replace the current image-level `robot_client` source copy with an installable
versioned `aess_robots` package as part of the package migration above.
Extract validation/shared message types before splitting `ros_mqtt_bridge.py`;
preserve MQTT topic shapes and binary protocol with compatibility tests.
Add durable mission IDs/state and beacon retention as separate behavior changes,
not as incidental folder moves. No source-tree relocation was necessary for this
integration; this remains a staged refactor proposal.
