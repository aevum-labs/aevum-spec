# Aevum Protocol Specification
## Section 04: Architecture

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 4.1 Layer Model

Aevum is organized into four layers, each depending only on layers below it:

    ┌─────────────────────────────────────────────────────┐
    │  Layer 4: Consumers                                  │
    │  (AI agents, applications, humans, MCP tools)        │
    ├─────────────────────────────────────────────────────┤
    │  Layer 3: Five Functions                             │
    │  ingest · query · review · commit · replay           │
    ├─────────────────────────────────────────────────────┤
    │  Layer 2: Kernel Services                            │
    │  barriers · consent · policy · audit · graph        │
    ├─────────────────────────────────────────────────────┤
    │  Layer 1: Storage                                    │
    │  GraphStore (Oxigraph / PostgreSQL / Jena)           │
    └─────────────────────────────────────────────────────┘

Consumers interact exclusively with Layer 3. They MUST NOT call Layer 2
or Layer 1 directly. The five functions are the complete public surface.

### 4.2 Request Flow

Every call to a function in Layer 3 MUST follow this sequence:

    1. Authentication — verify actor identity
    2. Barrier evaluation — all five absolute barriers, in order
    3. Consent check — active grant for this operation exists
    4. Policy evaluation — OPA + Cedar (parallel where possible)
    5. Graph operation — read or write to the knowledge graph
    6. Complication calls — invoke relevant complications in order
    7. Ledger append — record the AuditEvent
    8. OutputEnvelope assembly — assemble and return

Steps 1–4 MUST complete before step 5 begins. Steps 5–7 are atomic with
respect to the ledger: if step 6 or 7 fails, the ledger MUST record the
failure event, not silently drop it. Step 8 MUST always execute, even
when a previous step failed — the OutputEnvelope carries the failure status.

### 4.3 Three Named Graphs

Every GraphStore implementation MUST maintain exactly three named graphs:

    urn:aevum:knowledge    Working graph — typed entities and relationships
    urn:aevum:provenance   Immutable audit — append-only Merkle chain
    urn:aevum:consent      Consent ledger — OR-Set CRDT semantics

An implementation MUST NOT create additional named graphs at the kernel
level. Complications MAY maintain their own storage but MUST NOT write
to these three URIs directly.

### 4.4 Complication Architecture

Complications extend the kernel through a defined protocol (Section 11).
A complication is called during step 6 of the request flow. The kernel
passes a bounded context to each complication; the complication returns
data that is incorporated into the OutputEnvelope. Complications MUST NOT
access the storage layer directly. Complications MUST NOT call other
complications. The kernel manages all inter-complication coordination.

### 4.5 Deployment Modes

An Aevum node MAY be deployed in three modes:

**Embedded** — `aevum-core` imported as a Python library. No network
boundary between the kernel and its consumer. Suitable for single-process
applications.

**Server** — `aevum-server` exposes the five functions over HTTP at `/v1/`.
Any language can call the kernel. Suitable for multi-service architectures.

**Agent** — `aevum-mcp` exposes the five functions as MCP tools.
AI agents and assistants interact through the Model Context Protocol.
