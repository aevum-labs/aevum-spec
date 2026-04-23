# Aevum Protocol Specification
## Section 05: Output Envelope

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 5.1 Mandatory Return Type

Every call to `ingest`, `query`, `review`, `commit`, or `replay` MUST
return an `OutputEnvelope`. No function MAY return a raw value, None,
or use exception-raising as its primary control flow mechanism.

If a function encounters an error, it MUST return an `OutputEnvelope`
with `status="error"` and the error detail in the `data` field.

### 5.2 Field Specification

    status: Literal["ok", "error", "pending_review", "degraded", "crisis"]
        REQUIRED. The overall status of this operation.
        - "ok": operation completed successfully
        - "error": operation failed; detail in data field
        - "pending_review": operation paused awaiting human review
        - "degraded": operation completed but with reduced confidence
          due to complication unavailability or data gaps
        - "crisis": a crisis barrier was triggered; see Section 09

    data: dict[str, Any]
        REQUIRED. The primary payload of this operation.
        For "error" status: MUST contain "error_code" (str) and
        "error_detail" (str) keys.
        For "crisis" status: MUST contain "safe_message" (str) and
        "resources" (list[str]) keys.

    audit_id: str
        REQUIRED. The permanent ledger entry identifier for this
        operation. MUST be a UUIDv7 formatted as a URN:
        "urn:aevum:audit:<uuidv7>". MUST be unique across all time.

    confidence: float
        REQUIRED. Calibrated confidence in the data payload.
        MUST be in the range [0.0, 1.0] inclusive.
        MUST NOT be omitted even when confidence is high.
        1.0 is reserved for mechanically verifiable facts.
        For "error" or "crisis" status: MUST be 0.0.

    uncertainty: UncertaintyAnnotation
        REQUIRED. Structured annotation of uncertainty sources.
        MUST NOT be omitted. When confidence is 1.0, fields MAY
        be empty lists and empty strings.

        Fields:
          sources: list[str]
              Knowledge sources consulted during this operation.
          missing_context: list[str]
              Known gaps that affected the confidence score.
          assumptions: list[str]
              Assumptions made in the absence of complete data.
          confidence_basis: str
              Human-readable rationale for the confidence score.

    provenance: ProvenanceRecord
        REQUIRED. Chain of custody for the primary data payload.
        MUST NOT be omitted.

        Fields:
          source_id: str
              Identifier of the originating data source.
          ingest_audit_id: str
              audit_id of the ledger entry that ingested the
              source data. For `ingest` calls, this is the
              audit_id of this very OutputEnvelope.
          chain_of_custody: list[str]
              Ordered list of audit_ids showing data lineage
              from source to this response.
          classification: int
              Classification level of the primary payload.
              MUST be 0, 1, 2, or 3
              (see Section 02 and Section 09.3 for level definitions).
          model_id: str | None
              If the data was generated or transformed by an
              AI model, the model identifier. None otherwise.

    review_required: bool
        REQUIRED. True if and only if this operation has been
        paused at a `review` gate. MUST be consistent with
        status: if status is "pending_review" then
        review_required MUST be True.

    review_context: ReviewContext | None
        REQUIRED when review_required is True. MUST be None
        when review_required is False.

        Fields:
          proposed_action: str
              Human-readable description of the action requiring review.
          reason: str
              Why this action requires human review.
          deadline: str | None
              ISO 8601 datetime after which veto-as-default activates.
              None means no automatic veto.
          autonomy_level: int
              Autonomy level (1–5) of the requesting agent.
          risk_assessment: str
              Brief statement of consequence if the action is approved.

    source_health: SourceHealthSummary
        REQUIRED. Summary of complication availability during this call.

        Fields:
          available: list[str]
              Complication identifiers that responded normally.
          degraded: list[str]
              Complication identifiers with reduced output.
          unavailable: list[str]
              Complication identifiers that did not respond.
          overall: "healthy" | "degraded" | "critical"
              "healthy": all registered complications available
              "degraded": some complications unavailable or degraded
              "critical": majority of complications unavailable

    warnings: list[str]
        REQUIRED. Human-readable warning messages about this operation.
        MUST be an empty list when there are no warnings.
        MUST NOT be null or omitted.

    schema_version: str
        REQUIRED. The version of the OutputEnvelope schema used.
        Current value: "1.0". Implementations MUST reject envelopes
        with schema_version values they do not recognize.

    reasoning_trace: ReasoningTrace
        REQUIRED. Ordered record of reasoning steps taken.
        MUST NOT be omitted.

        Fields:
          steps: list[ReasoningStep]
              Each step has: step_id (str), description (str),
              inputs (list[str]), output_summary (str),
              duration_ms (int).
          total_duration_ms: int
              Wall-clock duration of the full operation in milliseconds.

### 5.3 Stability Guarantee

Once this specification reaches version 1.0, the fields defined in
Section 5.2 are frozen. New optional fields MAY be added in minor versions.
Existing fields MUST NOT be removed or have their types narrowed.
Consumers MUST ignore unknown fields to support forward compatibility.
