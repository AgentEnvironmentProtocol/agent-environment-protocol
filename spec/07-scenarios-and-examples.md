# 07 — Scenarios & Examples

This document walks through three end-to-end scenarios illustrating how the **Agent Environment Protocol (AEP)** formal grammar, transparent passthrough mounting, Control/Data Plane operations, and session roles behave in practice.

Related documents: [01 — Formal Grammar](./01-formal-grammar.md) | [02 — Tree Nodes & Data Model](./02-tree-nodes.md) | [03 — Passthrough & Composition](./03-passthrough-and-composition.md) | [04 — Control & Data Planes](./04-control-and-data-planes.md) | [06 — Profiles, Sessions & Vocabularies](./06-profiles-sessions-and-vocabularies.md)

---

## Scenario 1 — Desktop IDE with Mounted Embedded Terminal & Browser Webview

**Goal**: Inspect a workstation IDE that embeds an out-of-process terminal emulator and an embedded browser preview (`web.iframe`) directly inside structural layout panels, expand lazy DOM subtrees on demand, inspect a cropped screenshot region, and run a command in the terminal.

### Projected AEP Tree

```text
Root
 ├── head:
 │    ├── aep.protocolVersion: "1.0.0-draft"
 │    ├── aep.profiles: ["aep-core-streaming"]
 │    └── aep.vocabularies: ["aep.desktop", "aep.web"]
 └── body
      └── ou#workstation [type="desktop.host"]
           ├── channel#sys-audio [mediaType="audio/opus", direction="sense", deliveryMode="stream"]
           └── interface#ide [type="desktop.window", meta={aep.capture:{binary:true,refId:"bin-ide",regionSupported:true}}]
                └── structure#layout [type="desktop.panel"]
                     ├── input#editor [type="desktop.textField", mediaType="text/plain", value="fn main() {}", methods=[replaceRange, save]]
                     ├── interface#term [type="desktop.terminal"]         <-- transparently mounted terminal AEP server
                     │    ├── content#scrollback [type="desktop.textLog", mediaType="text/plain", temporalMode="dynamic"]
                     │    └── input#stdin [type="desktop.commandInput", mediaType="text/plain", methods=[sendLine]]
                     └── interface#preview [type="web.iframe"]             <-- transparently mounted browser tab AEP server
                          └── structure#dom-root [type="web.page", hasChildren=true]
```

### Interaction Sequence

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Agent Client
    participant Host as Desktop AEP Host
    participant Term as Mounted Terminal AEP Server
    participant Web as Mounted Webview AEP Server

    Agent->>Host: tree.query (rootPath="/workstation/ide", kinds=["interface", "input"])
    Host-->>Agent: Matches (/workstation/ide/layout/editor, .../term, .../preview)

    Agent->>Host: node.loadChildren (path="/workstation/ide/layout/preview/dom-root")
    Host->>Web: node.loadChildren (path="/dom-root")
    Web-->>Host: Child elements (web.header, web.form, web.button)
    Host-->>Agent: Rewritten children under .../preview/dom-root

    Agent->>Host: data.getBinary (refId="bin-ide", params={region: {x:0,y:0,width:600,height:400,coordSpace:"pixel"}})
    Host-->>Agent: Cropped PNG bytes

    Agent->>Host: node.invoke (path="/workstation/ide/layout/term/stdin", method="sendLine", params={text: "cargo test"})
    Host->>Term: node.invoke (path="/stdin", method="sendLine", params={text: "cargo test"})
    Term-->>Host: Result (ok)
    Term-->>Host: node.updated (path="/scrollback", value="test result: ok")
    Host-->>Agent: node.updated (path="/workstation/ide/layout/term/scrollback", value="test result: ok")
```

---

## Scenario 2 — Embodied Robot ("Body of Senses") + Cloud Workspace in One Client Tree

**Goal**: A single AEP client connects simultaneously to a physical robot AEP server (which itself transparently mounts an arm-controller AEP server) and a remote cloud workspace AEP server, assembling one unified tree of senses, actuators, and software tools.

### Client-Side Composite Tree

```text
Root
 ├── head:
 │    ├── aep.protocolVersion: "1.0.0-draft"
 │    ├── aep.profiles: ["aep-core-streaming"]
 │    └── aep.vocabularies: ["aep.robotics", "aep.desktop"]
 └── body
      ├── ou#robot [type="robotics.body", methods=[robotics.estop]]         <-- mounted from Robot AEP Server
      │    ├── ou#head [type="robotics.head"]
      │    │    ├── channel#left-eye  [type="robotics.camera",  mediaType="video/h264", direction="sense", deliveryMode="stream"]
      │    │    ├── channel#right-eye [type="robotics.camera",  mediaType="video/h264", direction="sense", deliveryMode="stream"]
      │    │    ├── channel#ears      [type="robotics.mic",     mediaType="audio/opus", direction="sense", deliveryMode="stream"]
      │    │    └── channel#voice     [type="robotics.speaker", mediaType="audio/opus", direction="act",   deliveryMode="stream"]
      │    └── ou#right-arm [type="robotics.limb"]                         <-- server-side passthrough mount from Arm Controller
      │         ├── channel#joints       [type="robotics.joint",   mediaType="application/x-joints",        direction="sense",  deliveryMode="stream"]
      │         ├── channel#cmd          [type="robotics.joint",   mediaType="application/x-joints",        direction="act",    deliveryMode="stream"]
      │         ├── channel#gripper-loop [type="robotics.gripper", mediaType="application/x-gripper-servo", direction="duplex", deliveryMode="stream"]
      │         └── interface#safety-hud [type="robotics.statusHud"]
      │              ├── content#torque-warning [mediaType="text/plain", temporalMode="dynamic", value="nominal"]
      │              └── input#estop [type="desktop.button", methods=[robotics.estop]]
      └── ou#cloud-workspace [type="desktop.host"]                         <-- mounted from Cloud Sandbox AEP Server
           └── interface#shell [type="desktop.terminal"]
                └── content#stdout [mediaType="text/plain", temporalMode="dynamic"]
```

### Interaction Sequence

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Agent Client (Unified Body)
    participant Robot as Robot AEP Server
    participant Arm as Arm AEP Sub-Server
    participant Cloud as Cloud Sandbox AEP Server

    Agent->>Robot: session.initialize + tree.get
    Agent->>Cloud: session.initialize + tree.get
    Note over Agent: Client mounts Robot under /robot and Cloud under /cloud-workspace

    Agent->>Robot: channel.open (path="/head/left-eye")
    Robot-->>Agent: channel.chunk (live H.264 vision sense frames)

    Agent->>Robot: channel.open (path="/right-arm/gripper-loop")
    Robot->>Arm: channel.open (path="/gripper-loop")
    Arm-->>Robot: subscriptionId + direction="duplex"
    Robot-->>Agent: subscriptionId + direction="duplex"

    Agent->>Robot: channel.send (subscriptionId, data=gripTargetBytes)
    Robot->>Arm: channel.send (subscriptionId, data=gripTargetBytes)
    Arm-->>Robot: channel.chunk (force feedback telemetry)
    Robot-->>Agent: channel.chunk (force feedback telemetry)

    Agent->>Cloud: node.invoke (path="/shell", method="run", params={cmd: "log-pick-event"})
    Cloud-->>Agent: Result (logged)
```

---

## Scenario 3 — Multi-User Shared Session (`controller` + `observer`) with `stream+seek`

**Goal**: An autonomous `controller` agent drives an environment while a human supervisor connects as an `observer`, sharing the fanned-out `"sense"` stream and inspecting historical frames via `channel.seek` / `channel.sample` without risking accidental writes.

### Interaction Sequence

```mermaid
sequenceDiagram
    autonumber
    participant Ctrl as Agent Client (role="controller")
    participant Obs as Human Viewer (role="observer")
    participant Server as AEP Server (policy="single-controller")

    Ctrl->>Server: session.initialize (session={role: "controller"})
    Server-->>Ctrl: Result (session.role="controller")

    Obs->>Server: session.initialize (session={role: "controller", allowDowngradeToObserver: true})
    Server-->>Obs: Result (session.role="observer")

    Ctrl->>Server: channel.open (path="/robot/head/left-eye")
    Obs->>Server: channel.open (path="/robot/head/left-eye")
    Note over Server: Single underlying capture loop fans out sense chunks to both Ctrl and Obs
    Server-->>Ctrl: channel.chunk (sense frames)
    Server-->>Obs: channel.chunk (sense frames)

    Obs->>Server: node.invoke (path="/robot/head", method="robotics.estop")
    Server-->>Obs: ErrorResponse (READ_ONLY_SESSION)

    Obs->>Server: channel.sample (path="/robot/head/blackbox", offset=45000, unit="ms")
    Server-->>Obs: Sampled historical frame at 45000 ms (read-only allowed)

    Obs->>Server: channel.open (path="/robot/head/blackbox")
    Obs->>Server: channel.seek (subscriptionId="sub-replay", offset=42000, unit="ms")
    Server-->>Obs: Seek acknowledged at 42000 ms
    Server-->>Obs: channel.chunk (replayed frames from 42000 ms)
```
