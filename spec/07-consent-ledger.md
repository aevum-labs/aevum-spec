# Aevum Protocol Specification
## Section 07: Consent Ledger

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 7.1 Consent as Precondition

Consent is checked as Barrier 3 (Section 09) before any graph operation.
An implementation MUST NOT perform `ingest`, `query`, or `replay` on data
about a subject without verifying that an active, unexpired consent grant
exists for the requesting actor, for the requested operation, for a
purpose consistent with the request.

### 7.2 Consent Grant Schema

    grant_id: str (UUIDv4)
        REQUIRED. Globally unique identifier for this grant.

    subject_id: str
        REQUIRED. Identifier of the data subject (whose data is covered).

    grantee_id: str
        REQUIRED. Identifier of the entity granted permission.

    operations: list[str]
        REQUIRED. Operations covered by this grant.
        MUST be a non-empty subset of:
        ["ingest", "query", "replay", "export"]
        An empty list is invalid.

    purpose: str
        REQUIRED. The specific purpose for which data may be used.
        MUST NOT be a generic value such as "any" or "all purposes".
        Implementations SHOULD validate that purpose is specific enough
        to be auditable.

    classification_max: int
        REQUIRED. Maximum classification level (0–3) this grant covers.
        See Section 02 (Terminology) for level definitions.
        An operation on data classified above this level MUST be rejected
        even if all other grant conditions are met.

    granted_at: str (ISO 8601 datetime)
        REQUIRED. When this grant was created.

    expires_at: str (ISO 8601 datetime)
        REQUIRED. When this grant expires. All grants MUST have an expiry date.
        An implementation MUST reject grant creation requests without
        an expiry. Expired grants MUST be treated as absent.

    authorization_ref: str | None
        The audit_id of the `review` event that authorized this grant.
        SHOULD be present for grants that required human authorization.
        MAY be None for programmatic grants where authorization is
        established through other means.

    revocation_status: "active" | "revoked" | "expired"
        REQUIRED. Current lifecycle status of this grant.
        "active": grant is in effect
        "revoked": grant was explicitly revoked before expiry
        "expired": grant passed its expires_at datetime

### 7.3 Grant Lifecycle

Grants transition through states as follows:

    (created) → active
    active → revoked  (explicit revocation by grantee or subject)
    active → expired  (expires_at datetime has passed)

A grant in "revoked" or "expired" status MUST be treated identically
to an absent grant for all permission checks.

Revocation MUST be recorded as a new AuditEvent in the episodic ledger.
Revocation is effective immediately upon recording.

### 7.4 OR-Set CRDT Semantics

The consent ledger uses OR-Set (Observed-Remove Set) CRDT semantics
for distributed consistency. This means:
- A grant that is present in any replica is treated as present.
- Revocation requires explicit tombstoning that propagates to all replicas.
- In case of conflict between a revocation and a concurrent grant,
  the revocation MUST take precedence (safety over availability).

### 7.5 Purpose Validation

Implementations SHOULD validate that the `purpose` field of a consent
grant is specific, auditable, and consistent with the operation being
authorized. The Cedar policy engine (Section 09) is the RECOMMENDED
mechanism for purpose validation.

A purpose is considered specific if it identifies:
- The category of analysis or use
- The intended consumer of the result
- The timeframe (if applicable)

Example of a valid purpose: "quarterly-fraud-detection-model-training"
Example of an invalid purpose: "analytics"

No perpetual grants are permitted. All grants MUST have an expires_at
value that is a finite datetime. An implementation MUST reject any grant
creation request where expires_at is absent, null, or set to a value
representing an unbounded or perpetual duration.
