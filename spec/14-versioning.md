# Aevum Protocol Specification
## Section 14: Versioning and Stability

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 14.1 Specification Versioning

This specification uses Semantic Versioning (SemVer):
- **Major version:** Breaking changes to normative behavior.
- **Minor version:** Backwards-compatible additions to normative behavior.
- **Patch version:** Clarifications and corrections with no behavioral change.

The current version is **0.1.0-draft**.
Version 1.0.0 is the first stable release.

### 14.2 Frozen Elements

Once this specification reaches version 1.0.0, the following are frozen.
They MUST NOT change in any minor or patch version. Changing them
requires a new major version.

- The names of the five functions
- All OutputEnvelope mandatory fields and their types
- All 16 AuditEvent field names and their types
- The five absolute barriers and their trigger conditions
- The three Named Graph URIs
- The conformance layer structure
- The HTTP API public surface URL prefix `/v1/`
- The HTTP API admin surface URL prefix `/_aevum/v1/`
- The error format (RFC 9457 `application/problem+json`)
- The `X-Aevum-Key` header name
- The `Idempotency-Key` header behavior

### 14.3 Evolution Policy

**Adding fields to OutputEnvelope:** Allowed in minor versions.
New fields MUST be optional. Consumers MUST ignore unknown fields.

**Adding new function behavior:** Allowed in minor versions if the
addition does not change the behavior of existing conformant implementations.

**Adding new HTTP endpoints:** Allowed in minor versions.

**Adding new problem types:** Allowed in minor versions.

**Changing existing field types:** Requires a major version.

**Removing fields:** Requires a major version.

**Changing HTTP URL structure:** A new `/v2/` surface must be introduced.
The `/v1/` surface MUST remain available for a minimum of 18 months
after `/v2/` is released before deprecation.

### 14.4 Security Patches

Absolute barrier definitions (Section 09.3) MAY be updated as security
patches outside the normal release process. Crisis indicator lists
(Barrier 1) are maintained as security-sensitive material and updated
on their own cadence. Security patches MUST be documented in CHANGELOG.md
with a clear description of the change and its motivation.

### 14.5 Deprecation

Deprecated features are announced in CHANGELOG.md.
Deprecated features MUST remain functional for a minimum of one major
version cycle before removal.
