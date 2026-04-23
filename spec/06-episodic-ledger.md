# Aevum Protocol Specification
## Section 06: Episodic Ledger

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 6.1 Append-Only Invariant

The episodic ledger MUST be append-only. An implementation MUST NOT
provide any mechanism — API, configuration, administrative command,
or storage-layer access — that deletes, modifies, or overwrites any
ledger entry. The Audit Immutability Barrier (Section 09, Barrier 4)
enforces this at the kernel level.

### 6.2 AuditEvent — 18 Required Fields

Every ledger entry carries exactly these 16 fields. All are REQUIRED.
An entry with any field absent is invalid and MUST be rejected on write
and treated as corrupt on read.

    event_id: str (UUIDv7)
        Time-ordered unique identifier for this ledger entry.
        MUST be a valid UUIDv7. Used for deduplication and
        time-ordered scan.

    episode_id: str (UUID)
        Groups all events produced by a single function call.
        All events within one `ingest`, `query`, `review`, `commit`,
        or `replay` call MUST share the same episode_id.

    sequence: int (uint64)
        Per-stream monotonically increasing counter.
        MUST be strictly greater than the sequence of the previous
        event in the same stream. Used for optimistic concurrency.

    event_type: str
        Dispatch key identifying the type of event.
        MUST follow the pattern: "<function>.<outcome>".
        Examples: "ingest.accepted", "query.complete",
        "review.approved", "review.vetoed", "commit.appended",
        "replay.complete", "barrier.triggered".

    schema_version: str
        Version of the AuditEvent schema. Current value: "1.0".
        Used for upcasting during replay across schema versions.

    valid_from: str (RFC 3339 datetime)
        The time from which this event was true in the domain.
        Distinct from system_time — represents the business validity
        period, not the transaction time.

    valid_to: str | None (RFC 3339 datetime)
        The time after which this event is no longer true.
        None means the event is still valid.

    system_time: int (64-bit HLC timestamp)
        Transaction time using a Hybrid Logical Clock.
        Provides causal ordering across distributed nodes.
        MUST be monotonically non-decreasing within a node.

    causation_id: str | None
        The event_id of the event that caused this event.
        None for events with no causal predecessor in the ledger.

    correlation_id: str | None
        Groups events belonging to the same saga or workflow.
        Corresponds to the X-Request-ID from the HTTP surface.

    actor: str
        Authenticated identity of the entity that triggered this event.
        MUST NOT be empty. MUST match the authenticated actor from
        the request.

    trace_id: str | None
        W3C Trace Context trace identifier for distributed tracing.

    span_id: str | None
        W3C Trace Context span identifier.

    payload: dict
        The event-specific data. Schema varies by event_type.
        MUST be a valid JSON object (not null, not an array).

    payload_hash: str
        SHA3-256 hash of the canonical JSON serialization of the
        payload field. Used to detect payload tampering.
        Canonical form: keys sorted, no insignificant whitespace.

    prior_hash: str
        SHA3-256 hash of the immediately preceding AuditEvent in
        this node's sigchain, computed over all 16 fields of that
        event. For the first event in a new node, MUST be the
        SHA3-256 of the string "aevum:genesis".

    signature: str
        Ed25519 signature over the concatenation of all preceding
        fields in this entry, including prior_hash.
        Encoded as base64url (RFC 4648 §5, no padding).

    signer_key_id: str
        Identifier of the Ed25519 key used to produce the signature.
        Used for key rotation — the verifier locates the public key
        by this identifier.

### 6.3 Replay Guarantee

An implementation MUST be able to reconstruct the exact OutputEnvelope
that was returned for any past operation, given its audit_id, by reading
only the episodic ledger. The reconstruction MUST produce byte-identical
results to the original response when the same inputs are provided.

This guarantee requires that:
- All complication outputs used in the original response are recorded
  in the payload of the relevant AuditEvent.
- The classification level, consent state, and policy decisions at the
  time of the original call are recorded.
- The graph state at the time of the original call is reconstructible
  from the append-only ledger entries preceding that event.

### 6.4 Episode Semantics

An episode groups all ledger events produced by a single function call.
The term is borrowed from cognitive neuroscience, where an episode is a
distinct, temporally-structured memory unit with a beginning and an end.
In Aevum, one call to `ingest` might produce multiple events (e.g.,
"ingest.barrier_check", "ingest.consent_check", "ingest.accepted").
All share the same episode_id.

### 6.5 Hybrid Logical Clock

The `system_time` field uses a Hybrid Logical Clock (HLC) combining:
- Physical time (wall-clock, millisecond precision)
- A logical counter incremented when physical time does not advance

HLC timestamps MUST be 64-bit integers encoded as:
  bits 63–16: milliseconds since Unix epoch
  bits 15–0:  logical counter

An implementation MUST ensure system_time is monotonically non-decreasing
across all events at a single node.
