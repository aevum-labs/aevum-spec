# Aevum Protocol Specification
## Section 13: Conformance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 13.1 Conformance Claims

An implementation MAY claim conformance with this specification.
A conformance claim MUST specify which layers (Section 13.2) the
implementation passes.

A "full conformance" claim requires passing all six layers.

### 13.2 Six Conformance Layers

**Layer 1 — Wire Format**
Validates that OutputEnvelope, AuditEvent, and ComplicationManifest
are encoded correctly per their JSON Schema definitions in `schemas/`.
An implementation MUST pass Layer 1 for all other layers to be meaningful.

**Layer 2 (Semantic) — Function Behavior**
Golden-fixture tests for all five functions. Each fixture specifies
an input, the expected OutputEnvelope structure, and the expected
AuditEvent type. An implementation MUST produce output consistent with
every fixture.

**Layer 2 (HTTP) — HTTP API Behavior**
Golden-fixture tests for the `aevum-server` HTTP endpoints against a
running instance. Tests cover all endpoints defined in Section 10.
Implementations of `aevum-server` MUST pass this layer.

**Layer 3 — Invariants**
Property-based tests using randomized inputs to verify four invariants:
1. Append-only: the ledger never loses or modifies entries.
2. Replay determinism: replaying the same audit_id twice returns
   identical results.
3. Commit atomicity: no partial commits; failure leaves no trace.
4. Policy bypass: no input combination causes a barrier to be skipped.

**Layer 4 — Policy**
Tests that OPA and Cedar produce correct decisions for a defined set of
policy scenarios. Uses the policy engines as an oracle to verify that
the kernel's policy bridge routes decisions correctly.

**Layer 5 — Agent Protocol**
Tests that agent state machine transitions follow Section 12.3.
Specifically: that an agent in PENDING_REVIEW cannot transition to
ACTING without an explicit "review.approved" AuditEvent.

### 13.3 Conformance Test Suite

The reference conformance suite is maintained at:
`github.com/aevum-labs/aevum-conformance`

It is available as a GitHub composite action:
`aevum-labs/conform@v0`

### 13.4 Anti-Implementation Requirement

Before Phase 3 ships, a deliberately broken implementation MUST be
run against the conformance suite. Each barrier and each invariant
MUST be violated one at a time. The conformance suite MUST fail on
each violation. If a violation does not cause failure, the relevant
test is inadequate and MUST be fixed before Phase 3 proceeds.

### 13.5 EARL Reports

Conformance results MUST be reported in EARL (Evaluation and Report
Language) format. The conformance runner generates EARL reports
that can be submitted to `github.com/aevum-labs/aevum-conformance`
as evidence of conformance.
