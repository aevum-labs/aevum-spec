# Aevum Protocol Specification
## Section 11: Complication Manifest

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 11.1 Manifest Schema

Every complication MUST provide a signed manifest at installation time.

    name: str
        REQUIRED. Unique identifier for this complication.
        MUST match the pattern: [a-z][a-z0-9-]*/[a-z][a-z0-9-]+
        Format: "<publisher>/<name>" e.g. "acme/customer-graph"

    version: str (SemVer)
        REQUIRED. Semantic version of this complication.

    description: str
        REQUIRED. Human-readable description of what this complication does.

    capabilities: list[str]
        REQUIRED. List of named capabilities this complication provides.
        Any change to this list triggers a reapproval cycle.

    classification_max: int (0–3)
        REQUIRED. Maximum classification level this complication may
        access. Any increase triggers a reapproval cycle.

    functions: list[str]
        REQUIRED. Which of the five functions this complication intercepts.
        MUST be a subset of ["ingest", "query", "review", "commit", "replay"].

    auth:
        REQUIRED.
        scopes_required: list[str]
            OAuth/OIDC scopes required by this complication.
            Any change triggers a reapproval cycle.
        public_key: str
            Ed25519 public key (base64url) used to verify this manifest.

    health_endpoint: str | None
        URL of the complication's health check endpoint.
        SHOULD be provided. Used by the kernel's circuit breaker.

    schema_version: str
        REQUIRED. Current value: "1.0".

### 11.2 Lifecycle States

```
DISCOVERED → PENDING → APPROVED → ACTIVE → SUSPENDED → DECOMMISSIONED
                ↓
            REJECTED
```

- **DISCOVERED:** Manifest received. Technical validation in progress (automatic).
- **PENDING:** Technical validation passed. Awaiting admin approval (human gate).
- **REJECTED:** Admin rejected. Permanent for this manifest version.
  A new version with changes restarts the cycle.
- **APPROVED:** Admin approved. Activation begins.
- **ACTIVE:** Live, receiving data from the kernel.
- **SUSPENDED:** Admin-suspended. Not receiving data. Audit trail preserved.
- **DECOMMISSIONED:** Removed from registry. Audit trail references preserved
  for replay of historical events.

### 11.3 Reapproval Triggers

Any change to the following manifest fields requires a full new approval cycle
from DISCOVERED, even if the complication is currently ACTIVE:

- `capabilities` (any addition, removal, or modification)
- `classification_max` (any change)
- `auth.scopes_required` (any addition, removal, or modification)

An implementation MUST detect these changes on manifest update and transition
the complication to DISCOVERED state, suspending it from receiving data
until the new manifest completes the approval cycle.

### 11.4 Manifest Signing

The manifest MUST be signed with an Ed25519 key.
The signature is computed over the canonical JSON serialization of the
manifest (keys sorted, no insignificant whitespace), excluding the
`signature` field itself.

An implementation MUST verify the manifest signature before the
complication enters PENDING state. A complication with an invalid
or absent signature MUST be rejected at DISCOVERED state.

### 11.5 Circuit Breaker

The kernel MUST implement a circuit breaker for each ACTIVE complication.
If a complication fails to respond within the configured timeout for
a configurable number of consecutive calls, the circuit breaker trips
and the complication is treated as UNAVAILABLE.

When a complication is UNAVAILABLE:
- The kernel MUST NOT block the calling function.
- The OutputEnvelope status MUST be "degraded".
- The complication MUST be listed in source_health.unavailable.

The circuit breaker MUST reset automatically after a configurable
recovery period. Manual reset is available via the admin API.
