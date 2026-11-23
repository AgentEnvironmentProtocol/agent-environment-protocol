# Agent Environment Protocol (AEP) Specification

The **Agent Environment Protocol (AEP)** defines a language-agnostic, transport-agnostic protocol built around a **modular passthrough I/O tree**. An AEP client connects to one or more AEP environment providers to assemble a single, navigable **body of senses and actuation**—spanning desktops, browsers, terminals, mobile devices, cloud services, and robotic hardware.

AEP governs the **Control Plane** (tree state, discovery, events, method invocation, and lightweight payload references) and **Data Plane** (discrete binary reads and continuous bidirectional I/O streams), while delegating transport and networking to underlying protocols such as QUIC, WebRTC, WebSocket, or local IPC streams. Control-plane messages prefer compact binary serialization (Protocol Buffers) with lossless JSON equivalence.

## Specification Index

| Document | Scope |
| --- | --- |
| [00 — Overview & Architecture](./spec/00-overview.md) | Protocol scope, "body of senses" model, plane boundaries, connection guarantees, and high-level architecture |
| [01 — Formal Grammar](./spec/01-formal-grammar.md) | Normative EBNF tree grammar, uniform node headers, path addressing, and static/semantic well-formedness invariants |
| [02 — Tree Nodes & Data Model](./spec/02-tree-nodes.md) | Normative semantics of `Root`, `Head`, `Body`, `OU`, `Interface`, `Element` (`structure` / `content` / `input`), `Channel`, `Meta`, methods, events, and references |
| [03 — Passthrough & Composition](./spec/03-passthrough-and-composition.md) | Transparent AEP-to-AEP subtree mounting, client multi-root aggregation, path/ref isolation, `Head` reconciliation, and fault isolation |
| [04 — Control & Data Planes](./spec/04-control-and-data-planes.md) | Wire envelope, session handshake, unified `tree.*`/`node.*` RPCs, notifications, parameterized `data.getBinary`, `channel.*` streams, and error codes |
| [05 — Transport & Interoperability](./spec/05-transport-and-interop.md) | Delegated transport guarantees, QUIC binding, WebRTC real-time media/telemetry, WebSocket/IPC bindings, and Protobuf/JSON serialization |
| [06 — Profiles, Sessions & Vocabularies](./spec/06-profiles-sessions-and-vocabularies.md) | Capability negotiation, `aep-core-static` and `aep-core-streaming` profiles, `controller`/`observer` roles, security modes, and open domain vocabularies |
| [07 — Scenarios & Examples](./spec/07-scenarios-and-examples.md) | End-to-end ASCII trees and sequence diagrams for desktop/IDE, embodied robotics + cloud workspace, and multi-user `stream+seek` sessions |

## Design Principles & Evolution

AEP refines earlier desktop-specific agent interface designs around five core architectural shifts:

- **Open Tree Over Fixed Tiers**: Arbitrary recursive `OU` containers, nestable `Interface`s (including inside structural UI elements), and open `NamespacedIdent` type tags replace fixed host/OS/app/process/UI hierarchies and closed widget enums.
- **Canonical Path Addressing**: Every node in `Body` is addressed by a single uniform `Path` (such as `/robot/head/left-eye`) rather than fixed multi-field address structs.
- **Clean Split Between Elements and Channels**: `Element`s represent discrete `structure`, `content`, and `input` UI nodes, while `Channel`s own all continuous and seekable I/O streams using explicit agent-centric polarity (`sense`, `act`, `duplex`).
- **Fractal Transparent Passthrough**: Any AEP subtree can mount another AEP provider transparently—allowing a single AEP protocol to unify client-to-server, server-to-app, and agent-to-device boundaries without auxiliary sub-protocols.
- **Unified Control & Parameterized Data Reads**: A small, uniform RPC surface (`tree.*`, `node.*`, `data.*`, `channel.*`) and structured binary fetch parameters replace per-tier APIs and ad-hoc query-string handles.

## Core Governance

See [AGENTS.md](./AGENTS.md) for repository authoring and collaboration rules.
