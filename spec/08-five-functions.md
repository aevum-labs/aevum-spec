# Aevum Protocol Specification
## Section 08: The Five Functions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 8.1 Overview

The five functions are the complete public surface of the Aevum kernel.
They are the only way to interact with the knowledge graph, the episodic
ledger, the consent ledger, and the policy engine.

Internal names used in code are shown for reference only. The function
names (`ingest`, `query`, `review`, `commit`, `replay`) are normative.

| Function | Internal name | Verb |
|---|---|---|
| ingest | RELATE | Moves data through the governed membrane |
| query | NAVIGATE | Traverses the graph for a declared purpose |
| review | GOVERN | Presents a proposed action for human decision |
| commit | REMEMBER | Appends an event to the episodic ledger |
| replay | (new) | Reconstructs a past decision faithfully |

### 8.2 VERIFY is not a function

Prior versions of this architecture included a VERIFY function.
VERIFY behavior — data validation, consistency checking, policy
pre-screening — is performed inline within each of the five functions.
It is not exposed as a standalone callable. This is not an omission.

### 8.3 ingest

**Purpose:** Move data through the governed membrane into the knowledge graph.

**Preconditions (all MUST be satisfied before graph write):**
1. Actor is authenticated.
2. All five absolute barriers pass (Section 09).
3. A consent grant exists for the actor, covering `"ingest"`, for the
   subject identified in the data, for the declared purpose.
4. A complete ProvenanceRecord is provided.
5. The data's classification level does not exceed the actor's clearance.

**Postconditions (all MUST hold after successful return):**
1. The data is present in `urn:aevum:knowledge`.
2. An AuditEvent with event_type "ingest.accepted" is appended to the ledger.
3. The returned OutputEnvelope has status "ok" and audit_id matching the
   ledger entry.

**Failure behavior:**
- Barrier failure: return OutputEnvelope with status "error" and
  event_type "barrier.triggered" in the ledger.
- Consent failure: return OutputEnvelope with status "error",
  error_code "consent_required".
- Provenance failure: return OutputEnvelope with status "error",
  error_code "provenance_required".

**Idempotency:** `ingest` with the same `Idempotency-Key` MUST return the
original OutputEnvelope without re-executing. The ledger MUST NOT gain
a duplicate entry.

### 8.4 query

**Purpose:** Traverse the knowledge graph to assemble context for a
declared purpose.

**Preconditions:**
1. Actor is authenticated.
2. All five absolute barriers pass.
3. A consent grant exists for the actor, covering `"query"`, for all
   subjects whose data may be returned, for the declared purpose.
4. The requested data classification does not exceed the actor's clearance.

**Postconditions:**
1. The returned OutputEnvelope contains context assembled from the graph.
2. Any data classified above the actor's clearance is redacted before assembly.
3. An AuditEvent with event_type "query.complete" is appended to the ledger.
4. If any registered complication is unavailable, status is "degraded" and
   the unavailable complication is listed in source_health.unavailable.

**Veto-as-default for query:** If a `review` gate is encountered during
graph traversal (e.g., a data item is flagged for human verification),
the query MUST pause, return status "pending_review", and await resolution.

### 8.5 review

**Purpose:** Present a proposed action to a human decision-maker before
execution proceeds.

**Preconditions:**
1. Actor is authenticated.
2. All five absolute barriers pass.
3. A ProposedAction is provided describing what needs review.

**Veto-as-default:**
If `deadline` is specified in the ReviewContext and no human response
is received before the deadline, the implementation MUST treat the
absence of response as a veto. The action MUST NOT proceed.
Silence is a veto. This is not configurable.

**Postconditions (on approve):**
1. An AuditEvent with event_type "review.approved" is appended.
2. The OutputEnvelope has status "ok" and review_required False.
3. The calling flow may proceed.

**Postconditions (on veto, explicit or timeout):**
1. An AuditEvent with event_type "review.vetoed" is appended.
2. The OutputEnvelope has status "error" and error_code "review_vetoed".
3. The calling flow MUST NOT proceed with the proposed action.

**Pending state:**
While awaiting human response, the OutputEnvelope has status
"pending_review" and review_required True.

### 8.6 commit

**Purpose:** Explicitly append an event to the episodic ledger.

The other four functions append ledger entries automatically as part of
their operation. `commit` allows callers to record arbitrary business
events that are not the direct result of a function call — for example,
recording that a human decision was made outside the system.

**Preconditions:**
1. Actor is authenticated.
2. All five absolute barriers pass.
3. The event_type provided does not collide with kernel-reserved types.
   Kernel-reserved types begin with: "ingest.", "query.", "review.",
   "commit.", "replay.", "barrier.", "policy.".
   Caller-provided types SHOULD use a namespaced prefix.

**Postconditions:**
1. An AuditEvent is appended to the ledger with the provided payload.
2. The returned OutputEnvelope has status "ok" and the audit_id of
   the new ledger entry.

**Idempotency:** `commit` with the same `Idempotency-Key` MUST return the
original OutputEnvelope without appending a duplicate ledger entry.

### 8.7 replay

**Purpose:** Reconstruct the exact OutputEnvelope that was returned for
a past operation, given its audit_id.

**Preconditions:**
1. Actor is authenticated.
2. The audit_id references an existing ledger entry.
3. A consent grant exists for the actor covering `"replay"` for all
   subjects whose data is present in the original response.

**Postconditions:**
1. The returned OutputEnvelope is byte-identical to the original
   response, with the exception of the `audit_id` field (which is
   the audit_id of this replay call, not the original).
2. An AuditEvent with event_type "replay.complete" is appended to
   the ledger, referencing the original audit_id via causation_id.

**Immutability:** Replay MUST NOT modify the knowledge graph or any
ledger entry. It is a read-only reconstruction.

**Determinism:** Two replay calls with the same audit_id and the same
actor clearance MUST produce identical OutputEnvelopes.
