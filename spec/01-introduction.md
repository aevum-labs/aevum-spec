# Aevum Protocol Specification
## Section 01: Introduction

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 1.1 Problem Statement

AI systems operating over real-world data face a set of problems that no
existing infrastructure solves together:

**Provenance loss.** Once data enters a system, its origin is typically
discarded. When a downstream model produces an incorrect output, it is often
impossible to trace which source data caused it.

**Replay impossibility.** When a decision is questioned later, the system
cannot reconstruct the exact state of knowledge that produced it. The graph
has changed, the model has been updated, the source has been modified.

**Consent opacity.** Data about individuals is ingested and queried without
a systematic record of who consented to what, for what purpose, with what
expiry. Compliance is asserted rather than demonstrated.

**Policy fragility.** Safety rules are scattered across application code,
prompt engineering, and human review processes. They can be bypassed by
changing prompts, switching models, or taking a different code path.

**Extension chaos.** When new data sources or capabilities are added, they
are typically bolted on without governance. A new integration can silently
change the behavior of existing queries.

### 1.2 Design Goals

Aevum addresses these problems through five design commitments:

**Replay-first.** Every operation is designed so that it can be faithfully
reconstructed at any future point. The episodic ledger records not just
what happened, but the full causal context that produced it.

**Consent-as-precondition.** No data about a subject can be ingested,
queried, or replayed without an active, unexpired consent grant. Consent
is checked before any graph operation, not after.

**Provenance-as-precondition.** No data can enter the knowledge graph
without a complete chain of custody. The origin of every datum is known
and verifiable.

**Policy-as-infrastructure.** Safety rules are not applied at the
application layer. They are enforced by the kernel before any data
reaches a complication or an AI consumer. Absolute barriers cannot be
overridden by any operator configuration.

**Extension-through-governance.** New data sources and capabilities are
added as complications. Every complication goes through an approval
lifecycle. Any change to a complication's capabilities requires a new
approval cycle.

### 1.3 Relationship to Existing Work

Aevum is not a database, an orchestration framework, or a compliance
reporting tool. It is a context kernel — infrastructure that other
systems integrate with.

The five functions (`ingest`, `query`, `review`, `commit`, `replay`)
are the complete public surface. Everything else — storage backends,
AI model adapters, identity federation, CLI tooling — is either a
complication or a consumer.

### 1.4 Document Conventions

- RFC 2119 keywords (MUST, SHOULD, MAY) denote normative requirements.
- Code examples are illustrative. Where a code example conflicts with
  normative prose, the prose takes precedence.
- Section numbers in cross-references refer to sections of this
  specification unless otherwise noted.
- "Implementation" refers to any software that claims conformance with
  this specification.
