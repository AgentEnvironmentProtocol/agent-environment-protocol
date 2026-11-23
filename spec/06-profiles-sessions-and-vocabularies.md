# 06 — Profiles, Sessions & Vocabularies

This document defines capability negotiation, standard conformance profiles (`aep-core-static` and `aep-core-streaming`), multi-user session roles and concurrency policies, security modes, and optional open domain vocabularies (`aep.desktop.*`, `aep.web.*`, `aep.robotics.*`).

Related documents: [00 — Overview](./00-overview.md) | [01 — Formal Grammar](./01-formal-grammar.md) | [02 — Tree Nodes & Data Model](./02-tree-nodes.md) | [03 — Passthrough & Composition](./03-passthrough-and-composition.md) | [04 — Control & Data Planes](./04-control-and-data-planes.md) | [05 — Transport & Interoperability](./05-transport-and-interop.md) | [07 — Scenarios & Examples](./07-scenarios-and-examples.md)

---

## 1. Capability Negotiation & Partial Profiles

Implementations MAY support a well-defined partial subset of AEP—such as non-streaming UI environments without `Channel`s—by declaring capabilities explicitly during `session.initialize` and exposing the negotiated result in `Head["aep.capabilities"]`:

| Capability Field | Type / Values | Semantics |
| --- | --- | --- |
| `profile` | `string` (optional) | Named conformance profile (e.g., `"aep-core-static"`, `"aep-core-streaming"`). |
| `nodeKinds` | Subset of `ou`, `interface`, `channel`, `structure`, `content`, `input` | Allowed `NodeKind`s in `Body`. |
| `channelDirections` | Subset of `sense`, `act`, `duplex` | Allowed `ChannelSpec.direction` polarities. |
| `deliveryModes` | Subset of `stream`, `stream+seek` | Allowed `ChannelSpec.deliveryMode` values. |
| `queries` | `{ treeQuery?: boolean, loadChildren?: boolean }` | Support for `tree.query` and lazy `node.loadChildren`. |
| `passthrough` | `{ enabled: boolean, maxDepth?: number }` | Support for transparent AEP-to-AEP subtree mounting. |
| `sessions` | `{ policy?: "single-controller" \| "multi-controller" \| "isolated", allowObserverStreams?: boolean }` | Multi-user role policy and observer stream fan-out behavior. |
| `dataPlane` | `{ binaryReferences: boolean, streamReferences: boolean, seek: boolean, sample: boolean }` | Active Data-Plane operations (`data.getBinary`, `channel.open`, `channel.seek`, `channel.sample`). |
| `security` | `{ mode: string, reserved?: boolean }` | Active security mode declared for the session. |
| `vocabularies` | `string[]` (optional) | Optional semantic vocabulary namespaces used in `Body` (e.g., `["aep.desktop", "aep.web", "aep.robotics"]`). |

### Normative Conformance Rules

1. **Explicit Capability Boundary**: Peers MUST declare active `nodeKinds`, `channelDirections`, `deliveryModes`, and Data-Plane features during `session.initialize`.
2. **Strict Out-of-Scope Rejection**: Operations, node projections, or channel directions outside negotiated capabilities MUST fail explicitly with error code `UNSUPPORTED_CAPABILITY`.
3. **No Silent Downgrade**: Implementations MUST NOT silently drop unsupported `Channel` nodes, streaming writes, or passthrough features while reporting success.

---

## 2. Standard Core Profiles

### 2.1 `aep-core-static`

The **`aep-core-static`** profile targets lightweight request/response and event-driven environments (such as static/dynamic desktop or web UI projection) without continuous `Channel` streams:

- **Enabled `NodeKind`s**: `ou`, `interface`, `structure`, `content`, `input` (`channel` is disabled).
- **Data Plane**: Discrete `BinaryRef` resolution via `data.getBinary` (including parameterized `region` crops and opt-in `aep.capture` snapshots). `channel.*` stream operations fail with `UNSUPPORTED_CAPABILITY`.
- **Passthrough**: Enabled for `ou`, `interface`, `structure`, `content`, and `input` subtrees.
- **Security**: Supports `security.mode = "unauthenticated"` for local/development sessions.

```json
{
  "profile": "aep-core-static",
  "nodeKinds": ["ou", "interface", "structure", "content", "input"],
  "channelDirections": [],
  "deliveryModes": [],
  "queries": {
    "treeQuery": true,
    "loadChildren": true
  },
  "passthrough": {
    "enabled": true
  },
  "dataPlane": {
    "binaryReferences": true,
    "streamReferences": false,
    "seek": false,
    "sample": false
  },
  "security": {
    "mode": "unauthenticated",
    "reserved": true
  }
}
```

### 2.2 `aep-core-streaming`

The **`aep-core-streaming`** profile enables the full AEP tree including continuous sensory and actuation `Channel`s, time-series `stream+seek`, random sampling, and multi-user observer fan-out:

```json
{
  "profile": "aep-core-streaming",
  "nodeKinds": ["ou", "interface", "channel", "structure", "content", "input"],
  "channelDirections": ["sense", "act", "duplex"],
  "deliveryModes": ["stream", "stream+seek"],
  "queries": {
    "treeQuery": true,
    "loadChildren": true
  },
  "passthrough": {
    "enabled": true
  },
  "sessions": {
    "policy": "single-controller",
    "allowObserverStreams": true
  },
  "dataPlane": {
    "binaryReferences": true,
    "streamReferences": true,
    "seek": true,
    "sample": true
  },
  "security": {
    "mode": "unauthenticated",
    "reserved": true
  }
}
```

---

## 3. Multi-User Session Roles & Concurrency

AEP supports concurrent multi-client connections against a shared environment by negotiating an explicit session role during `session.initialize`:

| Session Role | Permissions & Constraints |
| --- | --- |
| `controller` | Full interactive control: MAY traverse and query the tree, subscribe to `sense` / `duplex` channels, open and send on `act` / `duplex` channels, and invoke all `MethodDecl`s (`none`, `read`, and `write`). |
| `observer` | Read-only monitoring: MAY traverse and query the tree, invoke `MethodDecl`s explicitly annotated `sideEffects: "none"` or `"read"`, and subscribe to `"sense"` channels (and read-only output of `"duplex"` channels if `allowObserverStreams !== false`). MUST be rejected with `READ_ONLY_SESSION` when invoking `"write"` (or unannotated) methods, opening `"act"` channels, or calling `channel.send`. |

Servers advertise one of three concurrency policies in `aep.capabilities.sessions.policy`:

1. **`single-controller`** *(default for shared environments)*: At most one active `controller` holds mutating access. Subsequent `controller` requests downgrade to `observer` when `allowDowngradeToObserver !== false`, or fail `session.initialize` with `CONTROLLER_OCCUPIED`.
2. **`multi-controller`**: Multiple concurrent `controller` sessions MAY mutate state and send on `"act"`/`"duplex"` channels simultaneously.
3. **`isolated`**: Each connected session receives an isolated `Body` projection.

> **Shared Capture Fan-Out**: Servers SHOULD fan out shared `"sense"` `Channel` capture loops (such as desktop screen streams, cameras, microphones, or sensor telemetry) across all subscribed `controller` and `observer` sessions rather than duplicating hardware/OS capture per viewer.

---

## 4. Security Modes

Production AEP deployments rely on transport-level encryption and verifiable peer authentication as defined in [05 — Transport & Interoperability](./05-transport-and-interop.md).

To support local development, trusted in-memory tests, and incremental bring-up:

- Peers MAY negotiate `security: { "mode": "unauthenticated", "reserved": true }` during `session.initialize` (reflected in `Head["aep.security"]`).
- In `"unauthenticated"` mode, identity verification and fine-grained authorization are explicitly inactive for the session.
- Servers MUST reject any unrecognized or disallowed `security.mode` value during `session.initialize` with error code `UNSUPPORTED_AUTH_MODE`.

---

## 5. Optional Open Domain Vocabularies

> **Non-Mandatory Conventions**: The vocabularies below are optional naming conventions built strictly on open `TypeTag` (`node.type`) and `Meta` (`node.meta`) primitives. They are NOT closed core grammar rules; core AEP implementations handle all `TypeTag`s uniformly.

### 5.1 Desktop Vocabulary (`aep.desktop.*`)

Models workstation, OS, application, windowing, and terminal environments inside `OU`, `Interface`, `Element`, and `Channel` nodes:

- **Common `TypeTag`s**:
  - `OU`: `desktop.host`, `desktop.os`, `desktop.app`, `desktop.process`, `desktop.service`
  - `Interface`: `desktop.screen`, `desktop.window`, `desktop.terminal`, `desktop.dialog`
  - `StructureElement`: `desktop.panel`, `desktop.tabBar`, `desktop.tab`, `desktop.section`, `desktop.stack`, `desktop.table`, `desktop.row`, `desktop.column`, `desktop.list`, `desktop.keyval`
  - `ContentElement` / `InputElement`: `desktop.label`, `desktop.textLog`, `desktop.image`, `desktop.textField`, `desktop.button`, `desktop.checkbox`, `desktop.select`, `desktop.commandInput`
- **Metadata & Method Conventions**:
  - `aep.bounds`: `{ x, y, width, height, coordSpace: "pixel" | "normalized" }`
  - `aep.capture`: `{ binary: true, mediaType: "image/png", refId, regionSupported: true }` for opt-in screen/window/element snapshots via `data.getBinary({ refId, params: { region } })`.
  - `desktop.hwnd`, `desktop.pid`, `desktop.title`, `desktop.processName`, `desktop.minimized`: Optional domain keys in `node.meta`.
  - `table.query` (`sideEffects: "read"`): Standard server-side filtering, sorting, column projection, and pagination (`{ filters, sort, columns, offset, limit }`) attached to large `desktop.table` elements.

### 5.2 Web & Browser Vocabulary (`aep.web.*`)

Models browser sessions, tabs, DOM semantics, and nested cross-origin frames:

- **Common `TypeTag`s**: `web.browser`, `web.tab`, `web.page`, `web.iframe` (conventionally projected as a nested `Interface` inside a `StructureElement` or parent `Interface`), `web.article`, `web.header`, `web.paragraph`, `web.form`, `web.link`, `web.button`, `web.input`, `web.canvas`.
- **Metadata Conventions**: `web.url`, `web.title`, `web.origin`, `web.role`, `web.focused`.

### 5.3 Robotics & Embodied Senses Vocabulary (`aep.robotics.*`)

Models physical and simulated robotic platforms as a unified body of sensory and actuation nodes:

- **Common `TypeTag`s**:
  - `OU`: `robotics.body`, `robotics.head`, `robotics.limb`, `robotics.arm`, `robotics.base`
  - `Interface`: `robotics.statusHud`, `robotics.gripperControl`, `robotics.teleopPanel`
  - `Channel`: `robotics.camera`, `robotics.depth`, `robotics.lidar`, `robotics.mic`, `robotics.speaker`, `robotics.imu`, `robotics.joint`, `robotics.gripper`
- **Channel Media & Polarity Conventions**:
  - Sensory feeds (`direction: "sense"`): `video/h264`, `image/png`, `audio/opus`, `application/x-depth`, `application/x-lidar`, `application/x-imu`, `application/x-joints`.
  - Actuation streams (`direction: "act"`): `audio/opus` (speech/audio output), `application/x-joints`, `application/x-twist` (velocity/pose setpoints).
  - Closed-loop control (`direction: "duplex"`): `application/x-gripper-servo` (high-rate torque/force feedback paired with position/force commands).
- **Safety & Coordinate Metadata Conventions**:
  - `robotics.frameId`: Coordinate transform frame (e.g., `"base_link"`, `"head_left_optical"`).
  - `robotics.rateHz`: Nominal sensor/control loop frequency.
  - `robotics.estop` (`MethodDecl` with `sideEffects: "write"`): Standard emergency-stop method attached to top-level `robotics.body` or subsystem `OU`s.
