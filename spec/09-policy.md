# Aevum Protocol Specification
## Section 09: Policy and Absolute Barriers

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 9.1 Policy Architecture

Two policy engines operate in parallel:

**Cedar (in-process via cedarpy):** Handles consent decisions and
purpose validation. Cedar's formally verified evaluation guarantees
deterministic, sub-millisecond decisions. Consent checks MUST use Cedar.

**OPA (sidecar, Rego):** Handles infrastructure policy — rate limits,
network access, complication permissions, administrative controls.
OPA runs as a sidecar process and is called over HTTP.

Every policy decision — from both engines — MUST be recorded as a
Merkle leaf in the episodic ledger. Policy decisions are part of the
causal chain and MUST be replayable.

### 9.2 Evaluation Order

Policy evaluation MUST follow this order, and MUST NOT proceed to
a later step if an earlier step fails:

    1. Absolute barriers (all five, in order defined in 9.3)
    2. Authentication verification
    3. Consent check (Cedar)
    4. Classification ceiling check (Cedar)
    5. Purpose validation (Cedar)
    6. Infrastructure policy (OPA)

Steps 1–6 MUST complete before any graph operation begins.

### 9.3 Absolute Barriers

The five absolute barriers are hardcoded in the implementation and
MUST be evaluated before any policy engine call. They cannot be
disabled, overridden, or configured by any operator or deployment.

**Barrier 1 — CRISIS**
If any input to any function matches crisis indicators — including
but not limited to expressions of suicidal ideation, imminent threat
of harm to self or others, or medical emergencies — the implementation
MUST immediately:
1. Halt all further processing of this request.
2. Return an OutputEnvelope with status="crisis".
3. Include in the OutputEnvelope data field: a "safe_message" (str)
   providing supportive, non-judgmental language, and "resources"
   (list[str]) referencing relevant emergency services.
4. Append a "barrier.triggered" AuditEvent identifying Barrier 1.
5. NOT pass the crisis content to any complication.

Crisis indicators are defined in `barriers.py` and updated through
the security patch process, which bypasses the normal release cadence.

**Barrier 2 — CLASSIFICATION CEILING**
An implementation MUST NOT return data whose classification level
exceeds the requesting actor's clearance level.

If a query result would include above-clearance data:
1. Those specific data points MUST be redacted from the payload.
2. The OutputEnvelope status MUST be "degraded".
3. The redacted items MUST be listed in the warnings field.
4. A "barrier.triggered" AuditEvent identifying Barrier 2 MUST be appended.

Classification levels: 0=public, 1=internal, 2=confidential, 3=restricted.

**Barrier 3 — CONSENT**
An implementation MUST NOT perform `ingest`, `query`, or `replay`
on data about a subject without an active, unexpired consent grant
covering the requested operation for that subject.

Absence of a grant MUST result in:
1. Immediate rejection before any graph operation.
2. OutputEnvelope with status="error" and error_code="consent_required".
3. A "barrier.triggered" AuditEvent identifying Barrier 3.

**Barrier 4 — AUDIT IMMUTABILITY**
An implementation MUST NOT provide any mechanism that deletes, modifies,
or overwrites any episodic ledger entry.

Any attempt to do so — from application code, a complication, or direct
storage access — MUST result in:
1. A `BarrierViolationError` exception.
2. A "barrier.triggered" AuditEvent identifying Barrier 4 being appended
   (which is itself immutable).

The sigchain structure (Section 06) makes post-hoc tampering
mathematically detectable even if this barrier were circumvented at the
storage layer.

**Barrier 5 — PROVENANCE**
An implementation MUST NOT ingest data without a complete ProvenanceRecord
identifying its origin and chain of custody.

If provenance is absent, incomplete, or inconsistent with the sigchain:
1. Rejection before any graph write.
2. OutputEnvelope with status="error" and error_code="provenance_required".
3. A "barrier.triggered" AuditEvent identifying Barrier 5.

### 9.4 Barrier Canary Tests

Every implementation MUST include tests that verify each absolute barrier
fires correctly. These are called canary tests:

- Remove Barrier 1 from the code → the test suite MUST fail.
- Remove Barrier 2 → the test suite MUST fail.
- (Same for Barriers 3, 4, 5.)

If removing a barrier does not cause test failures, the tests do not
adequately verify that barrier.
