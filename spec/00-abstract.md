# Aevum Protocol Specification
## Section 00: Abstract

**Version:** 0.1.0-draft
**Status:** Draft
**Repository:** https://github.com/aevum-labs/aevum-spec
**License:** CC-BY-4.0 + OWFa 1.0.1

---

Aevum is a replay-first, policy-governed context protocol for AI systems
operating over sensitive, multi-source data.

It defines five functions — `ingest`, `query`, `review`, `commit`, and `replay`
— that together form a governed membrane between raw data sources and AI
consumers. Every operation is recorded in an append-only episodic ledger.
Every past decision can be reconstructed exactly as it appeared at the time
it was made. No data enters the system without provenance. No query executes
without consent. No action exceeds its authorized classification level.

This specification defines:

- The `OutputEnvelope` — the mandatory return type of all five functions
- The `AuditEvent` — the 16-field episodic ledger entry
- The consent grant schema and lifecycle
- The behavioral contracts for all five functions
- The five absolute barriers that no operator can override
- The HTTP API surface for non-Python consumers
- The complication (extension) manifest schema and lifecycle
- The agent protocol for AI agents operating through the membrane
- The six conformance layers and their certification requirements
- The versioning and stability guarantees

Aevum is infrastructure. It does not orchestrate agents, generate reports,
store passwords, or expose a general-purpose graph query interface.
See Section 03 (Non-Goals) for the normative scope fence.
