# 01 — Formal Grammar

This document defines the normative **Extended Backus-Naur Form (EBNF)** grammar and static/semantic well-formedness invariants for the **Agent Environment Protocol (AEP)** tree.

Related documents: [00 — Overview & Architecture](./00-overview.md) | [02 — Tree Nodes & Data Model](./02-tree-nodes.md) | [03 — Passthrough & Composition](./03-passthrough-and-composition.md) | [04 — Control & Data Planes](./04-control-and-data-planes.md)

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
   Agent Environment Protocol (AEP) — Core Tree Grammar (v1.0 Draft)
   ============================================================================ *)

(* --- Root & Top-Level Partitions --- *)
Root             ::= Head Body ;
Head             ::= Meta* ;
Body             ::= BodyChild* ;
BodyChild        ::= OU | Interface | Channel ;

(* --- Organizational & Interface Containers --- *)
OU               ::= NodeHeader OUChild* ;        (* NodeKind MUST be "ou" *)
OUChild          ::= OU | Interface | Channel ;

Interface        ::= NodeHeader InterfaceChild* ; (* NodeKind MUST be "interface" *)
InterfaceChild   ::= Element | Interface | Channel ;

(* --- Continuous / Seekable I/O Leaf --- *)
Channel          ::= NodeHeader ChannelSpec ;     (* NodeKind MUST be "channel" *)

(* --- Discrete & Structural Interface Elements --- *)
Element          ::= StructureElement
                   | ContentElement
                   | InputElement ;

StructureElement ::= NodeHeader StructureChild* ; (* NodeKind MUST be "structure" *)
StructureChild   ::= Element | Interface ;

ContentElement   ::= NodeHeader ContentSpec Element* ; (* NodeKind MUST be "content" *)
InputElement     ::= NodeHeader InputSpec Element* ;   (* NodeKind MUST be "input" *)

(* --- Uniform Node Header (Shared by All Body Nodes) --- *)
NodeHeader       ::= NodeId
                     NodeKind
                     TypeTag?
                     Meta*
                     MethodDecl*
                     EventDecl*
                     LazyFlag? ;

NodeKind         ::= "ou"
                   | "interface"
                   | "channel"
                   | "structure"
                   | "content"
                   | "input" ;

NodeId           ::= Segment ;
TypeTag          ::= NamespacedIdent ;
Meta             ::= Key Value ;

MethodDecl       ::= Name
                     Description?
                     InputSchema?
                     OutputSchema?
                     InvocationMode?
                     SideEffect
                     Meta* ;

EventDecl        ::= Name
                     Description?
                     PayloadSchema?
                     Meta* ;

InvocationMode   ::= "request-response" | "notification" ;
SideEffect       ::= "none" | "read" | "write" ;
LazyFlag         ::= Boolean ;                    (* Wire field: hasChildren *)

(* --- NodeKind-Specific Payloads & Descriptors --- *)
ChannelSpec      ::= MediaType Direction DeliveryMode StreamRef? StreamMeta? ;
Direction        ::= "sense" | "act" | "duplex" ;
DeliveryMode     ::= "stream" | "stream+seek" ;

ContentSpec      ::= MediaType TemporalState (InlineValue | BinaryRef | ChannelRef)? ;
InputSpec        ::= MediaType InputSchema? InlineValue? ChannelRef? ;
TemporalState    ::= "static" | "dynamic" ;

(* --- Data-Plane References --- *)
BinaryRef        ::= RefId MediaType SizeBytes? Meta* ;
StreamRef        ::= RefId MediaType FrameRate? SampleRate? ChannelCount? SeekSpec? Meta* ;
SeekSpec         ::= Seekable SeekUnit? DurationMs? StartOffset? EndOffset? ;
SeekUnit         ::= "ms" | "seconds" | "frames" | "bytes" | "samples" ;

(* --- Addressing & References --- *)
Path             ::= "/" | ("/" Segment)+ ;
ChannelRef       ::= Path ;

(* --- Schema, Value & Spec Field Aliases --- *)
Description      ::= String ;
InputSchema      ::= SchemaObject ;
OutputSchema     ::= SchemaObject ;
PayloadSchema    ::= SchemaObject ;
SchemaObject     ::= Object ;
StreamMeta       ::= Meta* ;
InlineValue      ::= Value ;
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

Context-free productions alone do not express uniqueness, acyclicity, or reference integrity. A valid AEP tree MUST additionally satisfy all eight invariants below:

### Invariant 1 — Sibling `NodeId` Uniqueness & Canonical `Path`
- Every direct child of `Body` MUST carry a distinct `NodeId` within `Body`.
- Every direct child of any node MUST carry a distinct `NodeId` within that parent.
- `Segment` MUST NOT equal `"."` or `".."`, preventing ambiguity with relative path traversal tokens.
- As a consequence, every node in `Body` has a single canonical absolute `Path` formed by joining ancestor `NodeId`s with `"/"` (with `"/"` denoting `Body` itself).

### Invariant 2 — Strict Tree Acyclicity
- `Body` MUST form a strict directed tree with zero cycles.
- No node MAY appear as an ancestor of itself—directly or across transparent passthrough mounts ([03 — Passthrough & Composition](./03-passthrough-and-composition.md)). Servers detecting cyclic mount graphs MUST reject or break the cycle with error code `MOUNT_CYCLE`.

### Invariant 3 — `NodeKind`–Production Alignment & Disjoint Element Kinds
- Every `NodeHeader` MUST match its structural production:
  - `OU` nodes MUST have `NodeKind = "ou"` and MAY contain only `OU`, `Interface`, or `Channel` children (never bare `Element`s).
  - `Interface` nodes MUST have `NodeKind = "interface"` and MAY contain `Element`, `Interface`, or `Channel` children (never `OU`s).
  - `StructureElement` nodes MUST have `NodeKind = "structure"` and MAY contain only `Element` or `Interface` children (never `OU`s or `Channel`s).
  - `ContentElement` nodes MUST have `NodeKind = "content"` and MAY contain only `Element` children.
  - `InputElement` nodes MUST have `NodeKind = "input"` and MAY contain only `Element` children.
- Element kinds (`structure`, `content`, `input`) are strictly mutually exclusive per node.

### Invariant 4 — `Channel` Leaf Constraint & Agent-Centric Polarity
- `Channel` nodes (`NodeKind = "channel"`) MUST be strict leaves: they MUST NOT carry child nodes and MUST NOT advertise `hasChildren = true`.
- `ChannelSpec.direction` MUST be interpreted strictly from the **agent client's perspective**:
  - `"sense"`: continuous data flows from environment to agent.
  - `"act"`: continuous data flows from agent to environment.
  - `"duplex"`: bidirectional closed-loop flow between agent and environment.

### Invariant 5 — Open `TypeTag` & `Meta` Extensibility
- Core AEP implementations MUST accept any syntactically valid `NamespacedIdent` as `TypeTag` and any `Key Value` pair in `Meta` without rejecting unknown vocabularies.
- Unrecognized `TypeTag`s and `Meta` keys MUST be preserved transparently across passthrough relays unless an explicit policy strips private internal keys.

### Invariant 6 — `ChannelRef` Resolution
- Whenever a `ContentSpec` or `InputSpec` supplies a `ChannelRef`, that `Path` MUST resolve to a node whose `NodeKind` is `"channel"` within the active tree visible to the caller (after any passthrough path rewriting).

### Invariant 7 — Lazy Subtree Contract
- A container node (`ou`, `interface`, `structure`, `content`, `input`) MAY set `LazyFlag` (`hasChildren = true`) while omitting some or all of its children in an initial projection.
- Clients materialize omitted descendants on demand via `node.loadChildren` ([04 — Control & Data Planes](./04-control-and-data-planes.md)).
- Setting `hasChildren = true` on a `Channel` node is invalid and MUST be rejected with `INVALID_MESSAGE`.

### Invariant 8 — Serialization Isomorphism
- Primary binary serialization (Protocol Buffers or equivalent typed binary schema) and optional JSON serialization MUST represent all grammar productions, `NodeKind` values, `Path` segments, `Value` types, and Data-Plane references losslessly and deterministically.
