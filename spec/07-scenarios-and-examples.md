# 07 — Scenarios & Examples

This document walks through four end-to-end scenarios illustrating how the **Agent Environment Protocol (AEP)** formal grammar, XAML-inspired node object model (`properties`, `state`, `methods`, `events`), composite `Component`s, rich `Document`s, transparent passthrough mounting, Control/Data Plane operations, and session roles behave in practice.

Related documents: [01 — Formal Grammar](./01-formal-grammar.md) | [02 — Tree Nodes & Data Model](./02-tree-nodes.md) | [03 — Passthrough & Composition](./03-passthrough-and-composition.md) | [04 — Control & Data Planes](./04-control-and-data-planes.md) | [06 — Profiles, Sessions & Vocabularies](./06-profiles-sessions-and-vocabularies.md)

---

## Scenario 1 — Desktop IDE with Mounted Embedded Terminal & Browser Webview

**Goal**: Inspect a workstation IDE window that embeds an out-of-process terminal emulator and an embedded browser page (`page`) directly inside a layout `grid`, expand lazy document subtrees on demand, inspect a cropped screenshot region, and run a command in the terminal.

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
           └── interface#ide [type="window", properties={title:"IDE", capture:{binary:true,refId:"bin-ide",regionSupported:true}}]
                └── structure#layout [type="grid", properties={columns:3}]
                     ├── editable#editor [type="textArea", properties={mediaType:"text/plain"}, state={value:"fn main() {}"}, methods=[setValue, replaceRange, save]]
                     ├── interface#term [type="desktop.terminal"]         <-- transparently mounted terminal AEP server
                     │    ├── view#scrollback [type="desktop.textLog", properties={mediaType:"text/plain", temporalMode:"dynamic"}]
                     │    └── editable#stdin [type="desktop.commandInput", properties={mediaType:"text/plain"}, methods=[setValue, sendLine]]
                     └── interface#preview [type="page"]                  <-- transparently mounted browser page AEP server
                          └── document#dom-root [properties={editable:false}, hasChildren=true]
```

### Interaction Sequence

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Agent Client
    participant Host as Desktop AEP Host
    participant Term as Mounted Terminal AEP Server
    participant Web as Mounted Webview AEP Server

    Agent->>Host: tree.query (rootPath="/workstation/ide", kinds=["interface", "editable"])
    Host-->>Agent: Matches (/workstation/ide/layout/editor, .../term, .../preview)

    Agent->>Host: node.loadChildren (path="/workstation/ide/layout/preview/dom-root")
    Host->>Web: node.loadChildren (path="/dom-root")
    Web-->>Host: Child elements (heading, form, button)
    Host-->>Agent: Rewritten children under .../preview/dom-root

    Agent->>Host: data.getBinary (refId="bin-ide", params={region: {x:0,y:0,width:600,height:400,coordSpace:"pixel"}})
    Host-->>Agent: Cropped PNG bytes

    Agent->>Host: node.invoke (path="/workstation/ide/layout/term/stdin", method="sendLine", params={text: "cargo test"})
    Host->>Term: node.invoke (path="/stdin", method="sendLine", params={text: "cargo test"})
    Term-->>Host: Result (ok)
    Term-->>Host: node.updated (path="/scrollback", state={value:"test result: ok"})
    Host-->>Agent: node.updated (path="/workstation/ide/layout/term/scrollback", state={value:"test result: ok"})
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
      │              ├── view#torque-warning [type="text", properties={mediaType:"text/plain", temporalMode:"dynamic"}, state={value:"nominal"}]
      │              └── control#estop [type="button", properties={label:"Emergency Stop"}, methods=[invoke, robotics.estop]]
      └── ou#cloud-workspace [type="desktop.host"]                         <-- mounted from Cloud Sandbox AEP Server
           └── interface#shell [type="desktop.terminal"]
                └── view#stdout [type="text", properties={mediaType:"text/plain", temporalMode:"dynamic"}]
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

---

## Scenario 4 — Composite Video Player `Component` & Rich `Document` with Embedded Fillable `Form`

**Goal**: An agent inspects an operator training `window` containing a two-column `grid`: on the left, a self-contained `media.videoPlayer` `Component` that owns seekable video/audio `Channel`s bound to its `view` viewport alongside standard playback primitives; on the right, a XAML-style rich `Document` combining prose blocks, tabular data, and an embedded interactive `form` whose `editable` fields the agent completes and submits.

### Projected AEP Tree

```text
Root
 ├── head:
 │    ├── aep.protocolVersion: "1.0.0-draft"
 │    └── aep.profiles: ["aep-core-streaming"]
 └── body
      └── interface#training-win [type="window", properties={title:"Operator Training Module"}, state={windowState:"normal", focused:true}]
           └── structure#main-split [type="grid", properties={columns:2}]
                ├── component#player [type="media.videoPlayer", state={playbackState:"paused", positionMs:0}, methods=[play, pause, seek]]
                │    ├── channel#video-out [mediaType="video/h264", direction="sense", deliveryMode="stream+seek", streamRef={refId:"vid-1", seek:{seekable:true, unit:"ms", durationMs:120000}}]
                │    ├── channel#audio-out [mediaType="audio/opus", direction="sense", deliveryMode="stream+seek", streamRef={refId:"aud-1", seek:{seekable:true, unit:"ms", durationMs:120000}}]
                │    ├── view#viewport [type="videoSurface", state={channelRef:"/training-win/main-split/player/video-out"}]
                │    └── structure#transport [type="stack", properties={orientation:"horizontal"}]
                │         ├── control#play-pause [type="button", properties={label:"Play"}, methods=[invoke]]
                │         ├── editable#scrub [type="slider", properties={valueSchema:{type:"number", minimum:0, maximum:120000}}, state={value:0}, methods=[setValue]]
                │         ├── editable#subtitles-toggle [type="checkbox", properties={label:"Captions"}, state={value:true}, methods=[setValue, toggle]]
                │         └── view#timecode [type="text", properties={mediaType:"text/plain", temporalMode:"dynamic"}, state={value:"00:00 / 02:00"}]
                └── document#module-guide [properties={title:"Safety & Calibration Guide", editable:false}]
                     ├── structure#h1 [type="heading", properties={level:1}]
                     │    └── view#title [type="text", state={value:"Calibration Checklist"}]
                     ├── structure#intro [type="paragraph"]
                     │    ├── view#lead [type="span", state={value:"Review the telemetry thresholds below, or consult the "}]
                     │    └── control#ref-link [type="link", properties={label:"Reference Manual"}, methods=[invoke]]
                     ├── structure#metrics [type="table"]
                     │    ├── structure#col-param [type="column", properties={label:"Parameter"}]
                     │    ├── structure#col-limit [type="column", properties={label:"Limit"}]
                     │    └── structure#row-1 [type="row"]
                     │         ├── structure#cell-1a [type="cell"]
                     │         │    └── view#v1 [type="text", state={value:"Max Torque"}]
                     │         └── structure#cell-1b [type="cell"]
                     │              └── view#v2 [type="text", state={value:"18.5 Nm"}]
                     └── structure#check-form [type="form", properties={title:"Verification Check"}, state={valid:false, dirty:false}, methods=[submit, reset, validate]]
                          ├── editable#answer [type="radioGroup", properties={label:"Confirm calibration step", required:true, valueSchema:{enum:["verified","needs-service"]}}, state={value:null}, methods=[setValue]]
                          ├── editable#notes [type="textField", properties={label:"Inspector Notes", placeholder:"Optional remarks"}, state={value:""}, methods=[setValue]]
                          └── control#submit [type="button", properties={label:"Submit Check"}, methods=[invoke]]
```

### Interaction Sequence

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Agent Client
    participant Host as Training App AEP Server

    Agent->>Host: channel.open (path="/training-win/main-split/player/video-out")
    Host-->>Agent: subscriptionId="sub-vid", direction="sense"
    Host-->>Agent: channel.chunk (initial H.264 frame at 0 ms)

    Agent->>Host: node.invoke (path="/training-win/main-split/player/transport/scrub", method="setValue", params={value: 45000})
    Host-->>Agent: Result (ok)
    Agent->>Host: node.invoke (path="/training-win/main-split/player", method="play")
    Host-->>Agent: Result (ok)
    Host-->>Agent: node.updated (path="/training-win/main-split/player", state={playbackState:"playing", positionMs:45000})
    Host-->>Agent: node.updated (path="/training-win/main-split/player/transport/timecode", state={value:"00:45 / 02:00"})
    Host-->>Agent: channel.chunk (video frames playing from 45000 ms)

    Agent->>Host: node.invoke (path="/training-win/main-split/module-guide/check-form/answer", method="setValue", params={value: "verified"})
    Host-->>Agent: node.updated (path="/training-win/main-split/module-guide/check-form/answer", state={value:"verified", valid:true, dirty:true})
    Agent->>Host: node.invoke (path="/training-win/main-split/module-guide/check-form/notes", method="setValue", params={value: "Nominal torque"})
    Host-->>Agent: node.updated (path="/training-win/main-split/module-guide/check-form", state={valid:true, dirty:true})

    Agent->>Host: node.invoke (path="/training-win/main-split/module-guide/check-form/submit", method="invoke")
    Host-->>Agent: Result ({submitted: true, status: "recorded"})
    Host-->>Agent: node.updated (path="/training-win/main-split/module-guide/check-form", state={valid:true, dirty:false})
```
