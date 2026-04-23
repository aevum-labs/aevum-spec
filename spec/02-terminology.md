# Aevum Protocol Specification
## Section 02: Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### Normative Definitions

**absolute barrier**
A hardcoded, unconditional, non-configurable stop that an implementation
MUST enforce before any policy evaluation or complication call. Absolute
barriers cannot be disabled, overridden, or configured by any operator.
Five absolute barriers are defined in Section 09.

**actor**
The entity that initiated an operation. An actor may be a human user,
an AI agent, an automated system, or another Aevum node. Actors are
identified by their authenticated identity at the time of the operation.

**audit_id**
A permanent, globally unique identifier for an episodic ledger entry.
Once assigned, an audit_id MUST NOT be reused or reassigned.

**classification level**
An integer from 0 to 3 indicating the sensitivity of data:
0 = public, 1 = internal, 2 = confidential, 3 = restricted.
An implementation MUST NOT return data at a classification level
above the requesting actor's clearance.

**complication**
An extension to the Aevum kernel that provides additional data sources,
capabilities, or output transformations. A complication MUST be approved
by an operator before it becomes active. The term is analogous to a
complication in a mechanical watch — a module that adds capability
while respecting the movement's constraints.

**consent grant**
A record in the consent ledger that authorizes a specific grantee to
perform specific operations on data about a specific subject, for a
specific purpose, until a specific expiry time. Consent grants are the
unit of permission in Aevum.

**controlled membrane**
The boundary enforced by the five functions. Data crosses the membrane
only through `ingest`. Queries cross outward only through `query`.
The membrane applies consent, provenance, policy, and barrier checks
to all crossings.

**episode**
A bounded sequence of related ledger events representing a single
coherent operation. All events produced by one call to any of the
five functions share an `episode_id`.

**episodic ledger**
The append-only, cryptographically signed record of all events that
have occurred in an Aevum node. An implementation MUST NOT delete,
modify, or overwrite any entry in the episodic ledger.

**governed membrane**
Synonym for controlled membrane. Preferred in architecture documentation.

**HLC**
Hybrid Logical Clock. A timestamp scheme that combines physical wall-clock
time with a logical counter to provide causally consistent ordering across
distributed nodes without requiring synchronized clocks.

**implementation**
Any software that claims conformance with this specification.

**ingest**
The function that moves data through the governed membrane into the
knowledge graph. Corresponds to the RELATE operation internally.

**knowledge graph**
The working graph (`urn:aevum:knowledge`) containing typed entities and
their relationships, assembled from ingested data and complication outputs.

**OutputEnvelope**
The mandatory return type of all five functions. Every call to `ingest`,
`query`, `review`, `commit`, or `replay` MUST return an OutputEnvelope.
No function may return a raw value, None, or raise an exception as its
primary control flow. Schema defined in Section 05.

**provenance**
The complete chain of custody for a datum — its origin, how it entered
the system, and all transformations applied to it.

**query**
The function that traverses the knowledge graph to assemble context for
a declared purpose. Corresponds to the NAVIGATE operation internally.

**replay**
The function that reconstructs the exact state of knowledge as it existed
at a specific past point in time, for a specific audit_id.

**review**
The function that presents a proposed action for human decision before
execution proceeds. Absence of human response within the deadline
(if specified) is treated as a veto. Corresponds to the GOVERN operation.

**commit**
The function that appends an event to the episodic ledger explicitly,
outside of the automatic logging performed by the other four functions.
Corresponds to the REMEMBER operation internally.

**sigchain**
The per-node append-only chain of Ed25519-signed ledger events. Each
event's `prior_hash` field contains the SHA3-256 hash of the immediately
preceding event, creating a tamper-evident chain.

**veto-as-default**
The principle that absence of human response to a `review` call is treated
as a veto, not as approval. Approval requires an explicit human action.
