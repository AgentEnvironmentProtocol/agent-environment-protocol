# Agent Instructions

## Role & Collaboration
- **Do not make assumptions** about requirements, design decisions, or intent.
- Your primary role is to **transcribe and co-author** the specification with the user.
- **Ask for approval** before introducing new ideas, design proposals, or structural changes.

## Repository & Document Format
- This is a **specification-only repository**: author **only Markdown** files.
- Specification documents MUST be **well-structured and unambiguous**, using RFC 2119 requirement keywords (`MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`, `MAY`, `OPTIONAL`) for normative clauses without adopting formal IETF RFC boilerplate.
- The specification is split across **multiple files** under `./spec/` and MUST maintain accurate relative cross-links between related documents and sections.
- Keep all terminology, grammar productions, method/notification names, and error codes strictly consistent across every document.

## Protocol Objective & Scope
- **Goal**: Define the **Agent Environment Protocol (AEP)** — a modular, passthrough I/O tree that enables autonomous agents to observe, reason over, and act across arbitrary target environments (including desktops, browsers, terminals, mobile devices, cloud workspaces, and robotics) as a unified body of senses and actuation.
- **Agnostic Design**: The protocol MUST remain **language-agnostic** and **transport-agnostic**.
- **Boundary of Concerns**: AEP governs the **Control Plane** and **Data Plane**, and delegates **transport, framing, encryption, and low-level networking** to underlying transport protocols.
