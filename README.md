# Agent Environment Protocol (AEP) Specification

**Specification Version:** `1.0.0-beta.0`

The **Agent Environment Protocol (AEP)** defines a language-agnostic, transport-agnostic protocol built around a **modular passthrough I/O tree**. An AEP client connects to one or more AEP environment providers to assemble a single, navigable **body of senses and actuation**—spanning desktops, browsers, terminals, headless programmatic APIs, mobile devices, cloud services, and robotic hardware.

AEP governs the **Control Plane** (tree state, discovery, events, method invocation, authentication, privilege elevation, and lightweight payload references) and **Data Plane** (discrete binary reads, multi-tier surface renders, and continuous binary or plain-event I/O streams), while delegating transport and networking to underlying protocols such as QUIC (TLS 1.3), WebRTC (`DTLS-SRTP`), WebSocket (`wss://`), or OS-authenticated local IPC streams. Control-plane messages prefer compact binary serialization (Protocol Buffers) with lossless JSON equivalence.

## Specification Index

| Document | Scope |
| --- | --- |
| [00 — Overview & Architecture](./spec/00-overview.md) | Protocol scope, "body of senses" model, plane boundaries, connection guarantees, and high-level architecture |
| [01 — Formal Grammar](./spec/01-formal-grammar.md) | Normative EBNF tree grammar, uniform node headers, path addressing, and static/semantic well-formedness invariants |
| [02 — Tree Nodes & Data Model](./spec/02-tree-nodes.md) | Normative semantics of `Root`, `Head`, `Body`, unified node objects (`properties`, `state`, `methods`, `events`, `meta`), `OU`, `Interface`, `Api` & `Endpoint`, `Element` (`structure`, `component`, `document`, `view`, `editable`, `control`), `Channel` (`binary` & `event`), multi-tier rendering (`renderMinimal`, `renderExpress`, `renderFull`), and Data-Plane references |
| [03 — Passthrough & Composition](./spec/03-passthrough-and-composition.md) | Transparent AEP-to-AEP subtree mounting, client multi-root aggregation, path/`uid`/`refId` isolation, `Head` reconciliation, credential boundaries, and distributed load-balanced failover |
| [04 — Control & Data Planes](./spec/04-control-and-data-planes.md) | Wire envelope (`seq`), session lifecycle (`initialize`, `authenticate`, `resume`, `elevate`, `dropElevation`, `acquireLease`, `releaseLease`), unified `tree.*`/`node.*` RPCs, notifications (`node.moved`, `session.leaseUpdated`), `data.getBinary`, `data.render`, `channel.*` binary/event streams, and error codes |
| [05 — Transport & Interoperability](./spec/05-transport-and-interop.md) | Delegated transport guarantees, QUIC+TLS 1.3 (`ALPN: "aep/1"`), WebRTC `DTLS-SRTP`, WebSocket (`wss://`), OS-authenticated IPC bindings, and Protobuf/JSON serialization |
| [06 — Profiles, Sessions & Vocabularies](./spec/06-profiles-sessions-and-vocabularies.md) | Protocol version negotiation, capability negotiation, `aep-core-static` and `aep-core-streaming` profiles, `controller`/`observer` roles, System 1/System 2 multi-agent leases & resumption, security modes, and open domain vocabularies (`aep.desktop.*`, `aep.web.*`, `aep.robotics.*`, `aep.api.*`) |
| [07 — Scenarios & Examples](./spec/07-scenarios-and-examples.md) | End-to-end ASCII trees and sequence diagrams for desktop/IDE, embodied robotics + cloud workspace, multi-user `stream+seek`, composite video player + forms, headless `Api` + elevation + `renderMinimal`, and multi-agent load-balanced cloud console |
| [08 — Security, Authentication & Privilege Elevation](./spec/08-security-and-elevation.md) | Layered transport & application auth, in-session credential rotation, `ELEVATION_REQUIRED` & `session.elevate`, normative agent elevation instructions, and passthrough credential isolation |
| [09 — Configuration & Passthrough Mount Manifests](./spec/09-configuration.md) | Canonical `aep.config.json` schema (with YAML/TOML equivalence), secret interpolation, transport listeners, reusable connections, and passthrough mount manifests |

## Design Principles & Evolution

AEP refines earlier desktop-specific agent interface designs around six core architectural shifts:

- **Open Tree Over Fixed Tiers**: Arbitrary recursive `OU` containers, nestable `Interface`s (`display`, `window`, `dialog`, `page`), headless programmatic `Api` & `Endpoint` trees (`service`, `rest`, `rpc`, `graphql`, `webhook`, `toolset`, `resource`, `collection`, `operation`), core layout containers (`form`, `section`, `stack`, `grid`, `table`, `column`, `row`, `cell`), and open `NamespacedIdent` type tags replace fixed host/OS/app/process/UI hierarchies and closed widget enums.
- **Canonical Path Addressing with Relocation Survival**: Every node in `Body` is addressed by a single uniform `Path` (such as `/robot/head/left-eye` or `/services/billing/invoices/inv-42`) rather than fixed multi-field address structs, with optional stable `uid`s, monotonic `revision`s, and `node.moved` notifications surviving dynamic shard/mount relocations.
- **XAML-Inspired Object Model & Clean Element/Channel Split**: Every node exposes uniform `properties`, live `state`, `methods`, `events`, and optional auxiliary `meta`. Discrete interface elements separate read-only presentation (`view`), stateful inputs (`editable`), stateless action triggers (`control`), rich interactive prose (`document`), and composite widgets (`component`) that can own and bind continuous/seekable or structured plain-event `Channel`s (`sense`, `act`, `duplex`; `payloadMode: "binary" | "event"`).
- **Fractal Transparent Passthrough, Load Balancing & Multi-Agent Concurrency**: Any AEP subtree can mount another AEP provider transparently via `aep.config.json` or behind many-to-many load-balanced gateways (`session.resume`), while dual-speed System 1 / System 2 agents coordinate safely via `Channel` streams, `session.acquireLease`, and optimistic `expectedRevision` checks.
- **Unified Control, Multi-Tier Rendering & Parameterized Data Reads**: A small, uniform RPC surface (`tree.*`, `node.*`, `data.*`, `channel.*`), server/client `renderMinimal`/`renderExpress`/`renderFull` fidelity tiers, and structured binary/event fetch parameters replace per-tier APIs and ad-hoc query-string handles.
- **Defense-in-Depth Security & Step-Up Elevation**: Transport cryptography (`QUIC+TLS 1.3`, `wss://`, `DTLS-SRTP`, OS peer credentials), application authentication (`bearer`, `mtls`, `os-peer`, `api-key`, `challenge-response`), and explicit scoped `session.elevate` grants protect high-impact operations without ambient-authority surprises.

## Core Governance

See [AGENTS.md](./AGENTS.md) for repository authoring and collaboration rules.
