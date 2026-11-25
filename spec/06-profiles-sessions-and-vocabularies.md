# 06 — Profiles, Sessions & Vocabularies

This document defines capability negotiation, standard conformance profiles (`aep-core-static` and `aep-core-streaming`), multi-user session roles and concurrency policies, security modes, and optional open domain vocabularies (`aep.desktop.*`, `aep.web.*`, `aep.robotics.*`, `aep.api.*`).

Related documents: [00 — Overview](./00-overview.md) | [01 — Formal Grammar](./01-formal-grammar.md) | [02 — Tree Nodes & Data Model](./02-tree-nodes.md) | [03 — Passthrough & Composition](./03-passthrough-and-composition.md) | [04 — Control & Data Planes](./04-control-and-data-planes.md) | [05 — Transport & Interoperability](./05-transport-and-interop.md) | [07 — Scenarios & Examples](./07-scenarios-and-examples.md) | [08 — Security, Authentication & Privilege Elevation](./08-security-and-elevation.md) | [09 — Configuration & Passthrough Mount Manifests](./09-configuration.md)

---

## 1. Capability Negotiation & Partial Profiles

### 1.1 Protocol Version Negotiation & Adaptation

- **Version Identifier Syntax**: Every AEP `protocolVersion` string MUST conform to Semantic Versioning 2.0.0 (`MAJOR.MINOR.PATCH` with optional `-prerelease` and `+build` identifiers). This specification defines protocol version **`"1.0.0-beta.0"`**.
- **Deterministic Selection Rules**:
  1. **Exact / Adapter Match First**: Given client `supportedVersions` (ordered by client preference) and server `supportedVersions` (ordered by server preference, including any explicit version adapters registered on the server), the server MUST first select the earliest entry in `client.supportedVersions` that matches an entry in `server.supportedVersions`.
  2. **Compatible SemVer Fallback (`MAJOR >= 1` Stable Releases)**: When neither candidate carries a `-prerelease` tag and both share the same `MAJOR` (`MAJOR >= 1`), peers with compatible-minor negotiation enabled MAY select the lower baseline version `MAJOR.min(MINOR_client, MINOR_server).PATCH` and gate newer minor capabilities through `Head["aep.capabilities"]`.
  3. **Pre-Release Isolation (`0.x` and `-prerelease`)**: Pre-release versions (such as `"1.0.0-beta.0"` vs. `"1.0.0-beta.1"` or `"1.0.0-draft"`) MUST NOT implicitly match across distinct pre-release tags unless an implementation explicitly configures a version adapter or compatibility entry for that version.
  4. **Rejection & Client Adaptation (`UNSUPPORTED_VERSION`)**: When no match exists, the server MUST reject `session.initialize` with `UNSUPPORTED_VERSION` and `error.data: { supportedVersions: string[] }`. Clients supporting multiple protocol versions or adapters MAY inspect `error.data.supportedVersions`, activate a compatible protocol version adapter, and retry `session.initialize`.

### 1.2 Capability Negotiation

Implementations MAY support a well-defined partial subset of AEP—such as non-streaming UI environments without `Channel`s—by declaring capabilities explicitly during `session.initialize` and exposing the negotiated result in `Head["aep.capabilities"]`:

| Capability Field | Type / Values | Semantics |
| --- | --- | --- |
| `profile` | `string` (optional) | Named conformance profile (e.g., `"aep-core-static"`, `"aep-core-streaming"`). |
| `nodeKinds` | Subset of `ou`, `interface`, `api`, `endpoint`, `channel`, `structure`, `component`, `document`, `view`, `editable`, `control` | Allowed `NodeKind`s in `Body`. |
| `channelDirections` | Subset of `sense`, `act`, `duplex` | Allowed `ChannelSpec.direction` polarities. |
| `deliveryModes` | Subset of `stream`, `stream+seek` | Allowed `ChannelSpec.deliveryMode` values. |
| `channelPayloadModes` | Subset of `binary`, `event` | Allowed `ChannelSpec.payloadMode` values (defaults to `["binary"]` if omitted). |
| `queries` | `{ treeQuery?: boolean, loadChildren?: boolean }` | Support for `tree.query` and lazy `node.loadChildren`. |
| `passthrough` | `{ enabled: boolean, maxDepth?: number }` | Support for transparent AEP-to-AEP subtree mounting. |
| `sessions` | `{ policy?: "single-controller" \| "multi-controller" \| "isolated", allowObserverStreams?: boolean, leases?: boolean, optimisticConcurrency?: boolean }` | Multi-user role policy, observer stream fan-out, cooperative subtree leases (`session.acquireLease`), and `expectedRevision` checks. |
| `resumption` | `{ enabled: boolean, replayWindowEvents?: number, sessionTtlMs?: number }` | Support for `session.resume`, outbound `seq` watermarks, and buffered stream replay across load-balanced reconnects. |
| `dataPlane` | `{ binaryReferences: boolean, streamReferences: boolean, seek: boolean, sample: boolean }` | Active Data-Plane operations (`data.getBinary`, `channel.open`, `channel.seek`, `channel.sample`). |
| `rendering` | `{ serverModes?: RenderMode[], formats?: string[], pathAnnotations?: boolean }` | Server-supported `data.render` modes (`"renderMinimal"`, `"renderExpress"`, `"renderFull"`), output MIME types, and `annotatePaths` support. |
| `security` | `{ mode: AuthMode, supportedModes?: AuthMode[], elevation?: boolean, reserved?: boolean }` | Active security mode, supported authentication mechanisms, and privilege elevation support. |
| `vocabularies` | `string[]` (optional) | Optional semantic vocabulary namespaces used in `Body` (e.g., `["aep.desktop", "aep.web", "aep.robotics", "aep.api"]`). |

### Normative Conformance Rules

1. **Explicit Capability Boundary**: Peers MUST declare active `nodeKinds`, `channelDirections`, `deliveryModes`, and Data-Plane features during `session.initialize`.
2. **Strict Out-of-Scope Rejection**: Operations, node projections, or channel directions outside negotiated capabilities MUST fail explicitly with error code `UNSUPPORTED_CAPABILITY`.
3. **No Silent Downgrade**: Implementations MUST NOT silently drop unsupported `Channel` nodes, streaming writes, or passthrough features while reporting success.

---

## 2. Standard Core Profiles

### 2.1 `aep-core-static`

The **`aep-core-static`** profile targets lightweight request/response and event-driven environments (such as static/dynamic desktop, web UI, or discrete headless `Api` projection) without continuous `Channel` streams:

- **Enabled `NodeKind`s**: `ou`, `interface`, `api`, `endpoint`, `structure`, `component`, `document`, `view`, `editable`, `control` (`channel` is disabled).
- **Data Plane**: Discrete `BinaryRef` resolution via `data.getBinary` (including parameterized `region` crops and opt-in `properties.capture` snapshots) and optional `data.render` modes. `channel.*` stream operations fail with `UNSUPPORTED_CAPABILITY`.
- **Passthrough**: Enabled for `ou`, `interface`, `api`, `endpoint`, `structure`, `component`, `document`, `view`, `editable`, and `control` subtrees.
- **Security**: Supports `security.mode = "unauthenticated"` for local/development sessions alongside production `AuthMode`s.

```json
{
  "profile": "aep-core-static",
  "nodeKinds": ["ou", "interface", "api", "endpoint", "structure", "component", "document", "view", "editable", "control"],
  "channelDirections": [],
  "deliveryModes": [],
  "channelPayloadModes": [],
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
  "rendering": {
    "serverModes": ["renderMinimal", "renderFull"],
    "formats": ["image/svg+xml", "image/png"],
    "pathAnnotations": true
  },
  "security": {
    "mode": "unauthenticated",
    "supportedModes": ["unauthenticated"],
    "elevation": false,
    "reserved": true
  }
}
```

### 2.2 `aep-core-streaming`

The **`aep-core-streaming`** profile enables the full AEP tree including continuous sensory, actuation, and structured event `Channel`s (on `Body`, `OU`, `Interface`, `Api`, `Endpoint`, or `ComponentElement`), time-series `stream+seek`, random sampling, multi-tier server rendering, privilege elevation, and multi-user observer fan-out:

```json
{
  "profile": "aep-core-streaming",
  "nodeKinds": ["ou", "interface", "api", "endpoint", "channel", "structure", "component", "document", "view", "editable", "control"],
  "channelDirections": ["sense", "act", "duplex"],
  "deliveryModes": ["stream", "stream+seek"],
  "channelPayloadModes": ["binary", "event"],
  "queries": {
    "treeQuery": true,
    "loadChildren": true
  },
  "passthrough": {
    "enabled": true
  },
  "sessions": {
    "policy": "single-controller",
    "allowObserverStreams": true,
    "leases": true,
    "optimisticConcurrency": true
  },
  "resumption": {
    "enabled": true,
    "replayWindowEvents": 1000,
    "sessionTtlMs": 300000
  },
  "dataPlane": {
    "binaryReferences": true,
    "streamReferences": true,
    "seek": true,
    "sample": true
  },
  "rendering": {
    "serverModes": ["renderMinimal", "renderExpress", "renderFull"],
    "formats": ["image/svg+xml", "image/png", "text/x-aep-layout"],
    "pathAnnotations": true
  },
  "security": {
    "mode": "bearer",
    "supportedModes": ["mtls", "os-peer", "bearer", "api-key", "challenge-response"],
    "elevation": true
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

### 3.1 Multi-Agent Collaboration, System 1 / System 2 Harnesses & Concurrency Guards

AEP's decoupled Control Plane and multiplexed Data Plane support both single-session dual-speed harnesses and multi-agent collaboration in one shared environment:

- **Dual-Speed ("System 1 / System 2") Processing**: Because `Channel` streams (`payloadMode = "binary"` or `"event"`) and `node.event` notifications flow asynchronously without blocking RPCs, an agent harness can route low-latency **System 1** reflex loops directly over `sense`/`act`/`duplex` channels while **System 2** planning loops perform `tree.query`, `data.render` (`renderMinimal`), and `node.invoke` concurrently.
- **Cooperative Subtree Leases (`session.acquireLease` / `session.releaseLease`)**: Under `multi-controller` policies, agents working across different subtrees or channels can claim an `"exclusive-write"` lease on a target subtree `Path` (or `uid`). Competing write attempts by other sessions fail cleanly with `LEASE_CONFLICT`, while read inspection and `"sense"` streams remain open to all authorized peers.
- **Optimistic Concurrency (`revision` / `expectedRevision`)**: When a fine-grained lock is unnecessary, callers MAY pass `expectedRevision` on `node.invoke` so stale writes fail cleanly with `REVISION_CONFLICT`.
- **Relocation & Failover Continuity**: When dynamic backend scaling moves a node (`node.moved`), callers holding a stable `uid` (with `followMoves = true`) or open `subscriptionId` survive path relocation and load-balancer failover via `session.resume` without interrupting active agents.

---

## 4. Security Modes

Production AEP deployments rely on transport-level encryption (`TLS 1.3`, `DTLS-SRTP`, or OS-isolated local IPC) paired with application-layer authentication and privilege elevation as normatively specified in [08 — Security, Authentication & Privilege Elevation](./08-security-and-elevation.md):

- Standardized `AuthMode` values are `"unauthenticated"`, `"mtls"`, `"os-peer"`, `"bearer"`, `"api-key"`, and `"challenge-response"`.
- Peers MAY negotiate `security: { "mode": "unauthenticated", "reserved": true }` for local development, trusted in-memory tests, and loopback bring-up.
- Servers MUST reject any unrecognized or disallowed `security.mode` value during `session.initialize` or `session.authenticate` with error code `UNSUPPORTED_AUTH_MODE`.

---

## 5. Core Built-In Types & Optional Domain Vocabularies

### 5.1 Core Built-In Types (Normative)

Single-segment un-namespaced `TypeTag`s are reserved by Core AEP so every client and agent understands standard surfaces, headless APIs, layouts, document blocks, and primitives without needing domain-specific extensions:

| `NodeKind` | Reserved Built-In `TypeTag`s |
| --- | --- |
| `interface` | `display`, `window`, `dialog`, `page` |
| `api` | `service`, `rest`, `rpc`, `graphql`, `webhook`, `toolset` |
| `endpoint` | `resource`, `collection`, `operation`, `streamEndpoint`, `webhookEndpoint` |
| `structure` | `form`, `section`, `stack`, `grid`, `table`, `column`, `row`, `cell`, `paragraph`, `heading`, `list`, `listItem`, `blockquote`, `codeBlock` |
| `view` | `text`, `span`, `label`, `badge`, `code`, `image`, `icon`, `videoSurface`, `audioOutput`, `canvas`, `progress` |
| `editable` | `textField`, `textArea`, `numberField`, `passwordField`, `searchField`, `checkbox`, `switch`, `toggleButton`, `radioGroup`, `radio`, `select`, `comboBox`, `slider`, `rangeSlider`, `dateField`, `colorField`, `voiceInput` |
| `control` | `button`, `iconButton`, `menuItem`, `link`, `command`, `toolbarButton` |

> **Open Domain Conventions**: The domain vocabularies below (`aep.desktop.*`, `aep.web.*`, `aep.robotics.*`, `aep.api.*`) are optional namespaced conventions built on `TypeTag` (`node.type`), `properties`, `state`, and `meta`. Core AEP implementations handle all valid `NamespacedIdent` `TypeTag`s uniformly.

### 5.2 Desktop Vocabulary (`aep.desktop.*`)

Models workstation, OS, application, windowing, and terminal environments inside `OU`, `Interface`, `Element`, and `Channel` nodes:

- **Common `TypeTag`s** (used alongside Core built-ins such as `window`, `dialog`, `grid`, `stack`, `table`, `button`, `textField`, `checkbox`, `select`):
  - `OU`: `desktop.host`, `desktop.os`, `desktop.app`, `desktop.process`, `desktop.service`
  - `Interface`: `desktop.terminal`, `desktop.tray`, `desktop.panelWindow`
  - `StructureElement`: `desktop.tabBar`, `desktop.tab`, `desktop.toolbar`, `desktop.keyval`
  - `ComponentElement`: `desktop.fileTree`, `desktop.codeEditor`
  - `ViewElement` / `EditableElement`: `desktop.textLog`, `desktop.commandInput`
- **Property, State & Metadata Conventions**:
  - `properties.bounds`: `{ x, y, width, height, coordSpace: "pixel" | "normalized" }`
  - `properties.capture`: `{ binary: true, mediaType: "image/png", refId, regionSupported: true }` for opt-in screen/window/element snapshots via `data.getBinary({ refId, params: { region } })`.
  - `properties.title`, `state.windowState` (`"normal" | "minimized" | "maximized" | "fullscreen"`), and `desktop.hwnd`, `desktop.pid`, `desktop.processName` in `node.meta`.
  - `table.query` (`sideEffects: "read"`): Standard server-side filtering, sorting, column projection, and pagination (`{ filters, sort, columns, offset, limit }`) attached to large `table` elements.

### 5.3 Web & Browser Vocabulary (`aep.web.*`)

Models browser sessions, tabs, DOM semantics, and nested cross-origin frames:

- **Common `TypeTag`s** (used alongside Core built-ins such as `page`, `dialog`, `document`, `section`, `heading`, `paragraph`, `form`, `link`, `button`, `textField`, `checkbox`, `select`):
  - `OU`: `web.browser`
  - `Interface`: `web.tab`, `web.iframe` (conventionally projected as a nested `Interface` inside a `StructureElement`, `ComponentElement`, `DocumentElement`, or parent `Interface`)
  - `StructureElement` / `ViewElement`: `web.article`, `web.nav`, `web.canvas`, `web.shadowRoot`
- **Property, State & Metadata Conventions**: `properties.title`, `state.focused`, and `web.url`, `web.origin`, `web.role` in `node.meta`.

### 5.4 Robotics & Embodied Senses Vocabulary (`aep.robotics.*`)

Models physical and simulated robotic platforms as a unified body of sensory and actuation nodes:

- **Common `TypeTag`s**:
  - `OU`: `robotics.body`, `robotics.head`, `robotics.limb`, `robotics.arm`, `robotics.base`
  - `Interface`: `robotics.statusHud`, `robotics.gripperControl`, `robotics.teleopPanel`
  - `ComponentElement`: `robotics.armModule`, `robotics.sensorPod` (owning local `Channel`s plus status `ViewElement`s and emergency `ControlElement`s)
  - `Channel`: `robotics.camera`, `robotics.depth`, `robotics.lidar`, `robotics.mic`, `robotics.speaker`, `robotics.imu`, `robotics.joint`, `robotics.gripper`
- **Channel Media & Polarity Conventions**:
  - Sensory feeds (`direction: "sense"`): `video/h264`, `image/png`, `audio/opus`, `application/x-depth`, `application/x-lidar`, `application/x-imu`, `application/x-joints`.
  - Actuation streams (`direction: "act"`): `audio/opus` (speech/audio output), `application/x-joints`, `application/x-twist` (velocity/pose setpoints).
  - Closed-loop control (`direction: "duplex"`): `application/x-gripper-servo` (high-rate torque/force feedback paired with position/force commands).
- **Safety & Coordinate Metadata Conventions**:
  - `robotics.frameId`: Coordinate transform frame in `node.meta` (e.g., `"base_link"`, `"head_left_optical"`).
  - `robotics.rateHz`: Nominal sensor/control loop frequency in `node.meta` or `node.properties`.
  - `robotics.estop` (`MethodDecl` with `sideEffects: "write"`): Standard emergency-stop method attached to top-level `robotics.body` or subsystem `OU`s, and optionally triggered via a `control` `button`.

### 5.5 Service & Cloud API Vocabulary (`aep.api.*`)

Models headless cloud services, REST/RPC gateways, tool providers, and webhook/event buses projected via `Api` and `Endpoint` nodes without coupling Core AEP to HTTP:

- **Common `TypeTag`s** (used alongside Core `Api`/`Endpoint` types `service`, `rest`, `rpc`, `graphql`, `webhook`, `toolset`, `resource`, `collection`, `operation`, `streamEndpoint`, `webhookEndpoint`):
  - `Api`: `api.openapi`, `api.grpc`, `api.mcpBridge`, `api.eventBus`
  - `Endpoint`: `api.queryEndpoint`, `api.mutationEndpoint`, `api.subscriptionEndpoint`
- **Metadata & Property Conventions (`meta` / `properties`)**:
  - `api.basePath`: Upstream base route prefix for bridged HTTP/REST services in `node.meta`.
  - `api.routeTemplate`: Underlying route template (e.g., `"/v1/invoices/{id}"`) in `node.meta`.
  - `api.httpMethod`: Originating HTTP verb (`"GET"`, `"POST"`, `"PATCH"`, `"DELETE"`) on bridged `MethodDecl.meta`.
  - `api.contentType`: Expected upstream payload media type (e.g., `"application/json"`).
  - `api.idempotent`: Boolean hint on `MethodDecl.meta` indicating safe retry semantics.
