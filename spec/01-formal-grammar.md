# 01 — Formal Grammar

This document defines the normative **Extended Backus-Naur Form (EBNF)** grammar and static/semantic well-formedness invariants for the **Agent Environment Protocol (AEP)** tree.

Related documents: [00 — Overview & Architecture](./00-overview.md) | [02 — Tree Nodes & Data Model](./02-tree-nodes.md) | [03 — Passthrough & Composition](./03-passthrough-and-composition.md) | [04 — Control & Data Planes](./04-control-and-data-planes.md) | [08 — Security, Authentication & Privilege Elevation](./08-security-and-elevation.md) | [09 — Configuration & Passthrough Mount Manifests](./09-configuration.md)

---

## 1. Notation Conventions

The grammar below uses an ISO/IEC 14977-style readable EBNF notation:

- `A ::= B ;` defines non-terminal `A` as production `B`.
- `A | B` denotes alternation (`A` or `B`).
- `A B` denotes sequence (`A` followed by `B`).
- `A?` denotes optionality (zero or one `A`).
- `A*` denotes zero or more repetitions of `A`.
- `A+` denotes one or more repetitions of `A`.
- `( ... )` groups sub-expressions.
- `"literal"` denotes a literal terminal value.
- `[A-Za-z0-9._-]` denotes a single character matching the specified character class.
- `(* comment *)` denotes a non-normative grammar annotation.

> **Abstract Syntax vs. Wire Encoding**: This grammar defines the **abstract syntax** of an AEP environment tree independent of serialization format. Implementations serialize these productions into Protocol Buffers or JSON as defined in [05 — Transport & Interoperability](./05-transport-and-interop.md).

---

## 2. Normative EBNF Grammar

```ebnf
(* ============================================================================
   Agent Environment Protocol (AEP) — Core Tree Grammar (v1.0.0-beta.0)
   ============================================================================ *)

(* --- Root & Top-Level Partitions --- *)
Root             ::= Head Body ;
Head             ::= Meta* ;
Body             ::= BodyChild* ;
BodyChild        ::= OU | Interface | Api | Channel ;

(* --- Organizational, Visual Interface & Headless API Containers --- *)
OU               ::= NodeHeader OUChild* ;        (* NodeKind MUST be "ou" *)
OUChild          ::= OU | Interface | Api | Channel ;

Interface        ::= NodeHeader InterfaceChild* ; (* NodeKind MUST be "interface" *)
InterfaceChild   ::= Element | Interface | Channel ;

Api              ::= NodeHeader ApiChild* ;       (* NodeKind MUST be "api" *)
ApiChild         ::= Endpoint | Api | Channel ;

Endpoint         ::= NodeHeader EndpointChild* ;  (* NodeKind MUST be "endpoint" *)
EndpointChild    ::= Endpoint | Channel ;

(* --- Continuous / Seekable / Event-Stream I/O Leaf --- *)
Channel          ::= NodeHeader ChannelSpec ;     (* NodeKind MUST be "channel" *)

(* --- Discrete, Structural, Composite & Document Interface Elements --- *)
Element          ::= StructureElement
                   | ComponentElement
                   | DocumentElement
                   | ViewElement
                   | EditableElement
                   | ControlElement ;

StructureElement ::= NodeHeader StructureChild* ; (* NodeKind MUST be "structure" *)
StructureChild   ::= Element | Interface ;

ComponentElement ::= NodeHeader ComponentChild* ; (* NodeKind MUST be "component" *)
ComponentChild   ::= Element | Interface | Channel ;

DocumentElement  ::= NodeHeader DocumentChild* ;  (* NodeKind MUST be "document" *)
DocumentChild    ::= Element | Interface ;

ViewElement      ::= NodeHeader Element* ;        (* NodeKind MUST be "view" *)
EditableElement  ::= NodeHeader Element* ;        (* NodeKind MUST be "editable" *)
ControlElement   ::= NodeHeader Element* ;        (* NodeKind MUST be "control" *)

(* --- Uniform Node Header / Object Anatomy (Shared by All Body Nodes) --- *)
NodeHeader       ::= NodeId
                     NodeUid?
                     NodeKind
                     TypeTag?
                     Revision?
                     Property*
                     State?
                     MethodDecl*
                     EventDecl*
                     Meta*
                     LazyFlag? ;

NodeKind         ::= "ou"
                   | "interface"
                   | "api"
                   | "endpoint"
                   | "channel"
                   | "structure"
                   | "component"
                   | "document"
                   | "view"
                   | "editable"
                   | "control" ;

NodeId           ::= Segment ;
NodeUid          ::= String ;
TypeTag          ::= NamespacedIdent ;
Revision         ::= NonNegInteger ;
Property         ::= Key Value ;
State            ::= StateField* BinaryRef? ChannelRef? ;
StateField       ::= Key Value ;
Meta             ::= Key Value ;

MethodDecl       ::= Name
                     Description?
                     InputSchema?
                     OutputSchema?
                     InvocationMode?
                     SideEffect
                     ElevationSpec?
                     Meta* ;

EventDecl        ::= Name
                     Description?
                     PayloadSchema?
                     Meta* ;

InvocationMode   ::= "request-response" | "notification" ;
SideEffect       ::= "none" | "read" | "write" ;
ElevationSpec    ::= ElevationLevel ScopeName* AllowedFactor* ReasonRequired? ;
ElevationLevel   ::= "none" | "confirm" | "elevated" | "admin" ;
ScopeName        ::= String ;
AllowedFactor    ::= "human-approval" | "totp" | "passkey" | "token" | "policy-attestation" ;
ReasonRequired   ::= Boolean ;
LazyFlag         ::= Boolean ;                    (* Wire field: hasChildren *)

(* --- NodeKind-Specific Descriptors --- *)
ChannelSpec      ::= MediaType Direction DeliveryMode PayloadMode? EventSchema? StreamRef? ;
Direction        ::= "sense" | "act" | "duplex" ;
DeliveryMode     ::= "stream" | "stream+seek" ;
PayloadMode      ::= "binary" | "event" ;
EventSchema      ::= SchemaObject ;

(* --- Data-Plane References --- *)
BinaryRef        ::= RefId MediaType SizeBytes? Meta* ;
StreamRef        ::= RefId MediaType FrameRate? SampleRate? ChannelCount? SeekSpec? Meta* ;
SeekSpec         ::= Seekable SeekUnit? DurationMs? StartOffset? EndOffset? ;
SeekUnit         ::= "ms" | "seconds" | "frames" | "bytes" | "samples" | "events" ;

(* --- Addressing & References --- *)
Path             ::= "/" | ("/" Segment)+ ;
ChannelRef       ::= Path ;

(* --- Schema, Value & Spec Field Aliases --- *)
Description      ::= String ;
InputSchema      ::= SchemaObject ;
OutputSchema     ::= SchemaObject ;
PayloadSchema    ::= SchemaObject ;
SchemaObject     ::= Object ;
SizeBytes        ::= NonNegInteger ;
FrameRate        ::= NonNegNumber ;
SampleRate       ::= NonNegNumber ;
ChannelCount     ::= PositiveInteger ;
Seekable         ::= Boolean ;
DurationMs       ::= NonNegNumber ;
StartOffset      ::= Number ;
EndOffset        ::= Number ;

(* --- Structured Value Model --- *)
Value            ::= Null | Boolean | Number | String | Array | Object ;
Array            ::= Value* ;
Object           ::= (String Value)* ;

(* --- Lexical Primitives --- *)
Segment          ::= SegmentChar+ ;
SegmentChar      ::= [A-Za-z0-9._-] ;
NamespacedIdent  ::= Segment ("." Segment)* ;
Key              ::= NamespacedIdent ;
Name             ::= NamespacedIdent ;
RefId            ::= String ;
MediaType        ::= String ;                     (* RFC 6838 MIME type or domain media type *)
Boolean          ::= "true" | "false" ;
Null             ::= "null" ;
String           ::= (* Unicode UTF-8 string *) ;
Number           ::= (* IEEE 754 finite decimal/floating-point number *) ;
NonNegNumber     ::= (* Number >= 0 *) ;
NonNegInteger    ::= (* Integer >= 0 *) ;
PositiveInteger  ::= (* Integer >= 1 *) ;
```

---

## 3. Static & Semantic Well-Formedness Invariants

Context-free productions alone do not express uniqueness, acyclicity, or reference integrity. A valid AEP tree MUST additionally satisfy all nine invariants below:

### Invariant 1 — Sibling `NodeId` Uniqueness, Canonical `Path`, Optional `NodeUid` & `Revision`
- Every direct child of `Body` MUST carry a distinct `NodeId` within `Body`.
- Every direct child of any node MUST carry a distinct `NodeId` within that parent.
- `Segment` MUST NOT equal `"."` or `".."`, preventing ambiguity with relative path traversal tokens.
- As a consequence, every node in `Body` has a single canonical absolute `Path` formed by joining ancestor `NodeId`s with `"/"` (with `"/"` denoting `Body` itself). Canonical `Path` remains the primary hierarchical address across all Control-Plane and Data-Plane operations.
- When present, `NodeUid` (`uid`) MUST be a non-empty string that is globally unique within the active projected `Body` tree. Unlike `Path`, a node's `uid` survives sibling renames, subtree re-parenting, and backend mount relocation (`node.moved`). Mount hosts MUST isolate downstream `uid` values across mounts so two mounted providers never collide.
- When present, `Revision` (`revision`) MUST increase monotonically whenever a node's `state` or `properties` mutate, supporting optimistic concurrency checks (`expectedRevision`).

### Invariant 2 — Strict Tree Acyclicity
- `Body` MUST form a strict directed tree with zero cycles.
- No node MAY appear as an ancestor of itself—directly or across transparent passthrough mounts ([03 — Passthrough & Composition](./03-passthrough-and-composition.md)). Servers detecting cyclic mount graphs MUST reject or break the cycle with error code `MOUNT_CYCLE`.

### Invariant 3 — `NodeKind`–Production Alignment, Disjoint Kinds & Action/State Split
- Every `NodeHeader` MUST match its structural production:
  - `OU` nodes MUST have `NodeKind = "ou"` and MAY contain only `OU`, `Interface`, `Api`, or `Channel` children (never bare `Element`s or `Endpoint`s).
  - `Interface` nodes MUST have `NodeKind = "interface"` and MAY contain `Element`, `Interface`, or `Channel` children (never `OU`s, `Api`s, or `Endpoint`s).
  - `Api` nodes MUST have `NodeKind = "api"` and MAY contain only `Endpoint`, nested `Api`, or `Channel` children (never `OU`s, `Interface`s, or `Element`s). `Api` nodes represent non-visual, non-physical programmatic surfaces and MUST NOT carry visual geometry or capture descriptors (`properties.bounds`, `properties.capture`).
  - `Endpoint` nodes MUST have `NodeKind = "endpoint"` and MAY contain only nested `Endpoint` or `Channel` children (never `OU`s, `Api`s, `Interface`s, or `Element`s). `Endpoint` nodes MUST NOT carry visual geometry or capture descriptors (`properties.bounds`, `properties.capture`).
  - `StructureElement` nodes MUST have `NodeKind = "structure"` and MAY contain only `Element` or `Interface` children (never `OU`s, `Api`s, `Endpoint`s, or `Channel`s).
  - `ComponentElement` nodes MUST have `NodeKind = "component"` and MAY contain `Element`, `Interface`, or `Channel` children (never `OU`s, `Api`s, or `Endpoint`s). A `ComponentElement` MAY own and control direct child `Channel`s and bind them to descendant `ViewElement` or `EditableElement` nodes via `State.channelRef`.
  - `DocumentElement` nodes MUST have `NodeKind = "document"` and MAY contain `Element` or `Interface` children (never `OU`s, `Api`s, `Endpoint`s, or `Channel`s).
  - `ViewElement` nodes MUST have `NodeKind = "view"` and MAY contain only `Element` children (never `OU`s, `Interface`s, `Api`s, `Endpoint`s, or `Channel`s). `ViewElement`s are read-only presentation primitives.
  - `EditableElement` nodes MUST have `NodeKind = "editable"` and MAY contain only `Element` children (never `OU`s, `Interface`s, `Api`s, `Endpoint`s, or `Channel`s). Every stateful input control whose state changes on interaction (such as text fields, checkboxes, switches, toggle buttons, selects, combo boxes, sliders, radio groups, and date/color fields) MUST use `NodeKind = "editable"`.
  - `ControlElement` nodes MUST have `NodeKind = "control"` and MAY contain only `Element` children (never `OU`s, `Interface`s, `Api`s, `Endpoint`s, or `Channel`s). `ControlElement`s are strictly stateless action/command triggers (such as buttons, icon buttons, menu items, and command links) and MUST NOT hold mutable input values in `State`.
  - `Channel` nodes MUST have `NodeKind = "channel"` and MUST carry `ChannelSpec`.
- All eleven `NodeKind` values—and specifically the six `Element` kinds (`structure`, `component`, `document`, `view`, `editable`, `control`)—are strictly mutually exclusive per node.

### Invariant 4 — `Channel` Leaf Constraint, Polarity & `PayloadMode`
- `Channel` nodes (`NodeKind = "channel"`) MUST be strict leaves: they MUST NOT carry child nodes and MUST NOT advertise `hasChildren = true`.
- `ChannelSpec.direction` MUST be interpreted strictly from the **agent client's perspective**:
  - `"sense"`: continuous data flows from environment to agent.
  - `"act"`: continuous data flows from agent to environment.
  - `"duplex"`: bidirectional closed-loop flow between agent and environment.
- `ChannelSpec.payloadMode` defaults to `"binary"` when omitted.
  - When `payloadMode = "binary"`, Data-Plane frames (`channel.chunk`, `channel.send`, `channel.sample`) carry raw binary payloads in `data` and MUST NOT carry structured `event` objects.
  - When `payloadMode = "event"`, Data-Plane frames carry structured `ChannelEvent` objects in `event` (validated against optional `ChannelSpec.eventSchema`) and MUST NOT carry `data`. Violations MUST fail with `PAYLOAD_MODE_MISMATCH`.

### Invariant 5 — Core Built-In `TypeTag`s & Open `Property` / `State` / `Meta` Extensibility
- Single-segment un-namespaced `TypeTag`s (containing no `"."`) are reserved for Core AEP built-in surface, layout, document, and primitive types defined in [02 — Tree Nodes & Data Model](./02-tree-nodes.md) and [06 — Profiles, Sessions & Vocabularies](./06-profiles-sessions-and-vocabularies.md).
- Core AEP implementations MUST accept any syntactically valid `NamespacedIdent` as `TypeTag` (including open namespaced types such as `prefix.name`) and any `Key Value` pair in `Property`, `State`, and `Meta` without rejecting unknown vocabularies.
- Unrecognized `TypeTag`s and `Property` / `State` / `Meta` keys MUST be preserved transparently across passthrough relays unless an explicit policy strips private internal `Meta` keys.

### Invariant 6 — `ChannelRef` Resolution
- Whenever a node's `State` supplies a `ChannelRef`, that `Path` MUST resolve to a node whose `NodeKind` is `"channel"` within the active tree visible to the caller (after any passthrough path rewriting).

### Invariant 7 — Lazy Subtree Contract
- Any container node (`ou`, `interface`, `api`, `endpoint`, `structure`, `component`, `document`, `view`, `editable`, `control`) MAY set `LazyFlag` (`hasChildren = true`) while omitting some or all of its children in an initial projection.
- Clients materialize omitted descendants on demand via `node.loadChildren` ([04 — Control & Data Planes](./04-control-and-data-planes.md)).
- Setting `hasChildren = true` on a `Channel` node is invalid and MUST be rejected with `INVALID_MESSAGE`.

### Invariant 8 — Serialization Isomorphism
- Primary binary serialization (Protocol Buffers or equivalent typed binary schema) and optional JSON serialization MUST represent all grammar productions, `NodeKind` values, `Path` segments, `Property` / `State` / `Meta` entries, `Value` types, and Data-Plane references losslessly and deterministically.

### Invariant 9 — Tabular `table` / `column` / `row` / `cell` Structural Integrity
- A `StructureElement` with `type = "table"` MUST only materialize direct children of `kind = "structure"` with `type = "column"` or `type = "row"`.
- A `StructureElement` with `type = "row"` MUST only materialize direct children of `kind = "structure"` with `type = "cell"`.
