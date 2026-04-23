# Aevum Protocol Specification
## Section 12: Agent Protocol

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 12.1 Autonomy Levels

Aevum uses the DeepMind taxonomy of autonomy levels, mapped to `review`
frequency requirements. Levels 1–5 have runtime representation.
Level 0 (no AI involvement) is defined for reference only; an
`AgentComplication` operating at L0 would be a contradiction in terms.

    L1 — TOOL:         AI suggests; human selects and executes
    L2 — CONSULTANT:   AI narrows options; human selects and executes
    L3 — COLLABORATOR: AI selects and executes; human can override
    L4 — EXPERT:       AI selects and executes; informs human
    L5 — AGENT:        AI selects and executes; human sets goals only

### 12.2 Review Frequency by Autonomy Level

| Level | review() requirement |
|---|---|
| L1 | MUST call review() before every action |
| L2–L3 | MUST call review() before consequential or irreversible actions |
| L4 | MUST call review() at session boundaries and on anomaly detection |
| L5 | Post-hoc audit only via replay(); MUST undergo constitutional review before L5 is enabled |

### 12.3 Agent State Machine

Observable states at the protocol level:

```
IDLE → PLANNING → ACTING → PENDING_REVIEW → ACTING (resumed)
                                          → SUSPENDED
                         → SUSPENDED
                         → TERMINATED
IDLE → TERMINATED
```

- **IDLE:** No active task.
- **PLANNING:** Internal reasoning. Opaque to the protocol.
- **ACTING:** Executing actions within policy.
- **PENDING_REVIEW:** Awaiting human `review()` before proceeding.
  The agent MUST NOT take further actions until review resolves.
- **SUSPENDED:** Admin-halted. State preserved. Resumption requires
  explicit admin action.
- **TERMINATED:** Task complete or cancelled.

State transitions MUST be recorded as AuditEvents in the episodic ledger.
An implementation MUST NOT allow an agent to transition from
PENDING_REVIEW to ACTING without an explicit "review.approved" AuditEvent.

### 12.4 AgentManifest

An `AgentComplication` is a `Complication` subtype with additional
manifest fields:

    autonomy_level: int (1–5)
        REQUIRED. Declared autonomy level. This sets the review frequency.
        Any change triggers reapproval (Section 11.3 rules apply).

    max_session_duration: int (seconds)
        REQUIRED. Maximum duration of a single agent session.
        After this duration, the agent MUST be terminated.

    allowed_functions: list[str]
        REQUIRED. Which of the five functions this agent may call.

    constitutional_review_required: bool
        REQUIRED. If True, this agent operates at L5 and has undergone
        constitutional review. MUST be False for L1–L4 agents.

### 12.5 Constitutional Review

Enabling L5 (fully autonomous) operation MUST require constitutional review:
a formal evaluation by human operators of the agent's goals, constraints,
decision-making process, and failure modes, recorded in the episodic ledger
as a special AuditEvent with event_type "agent.constitutional_review".

An implementation MUST NOT allow an agent with `autonomy_level=5` to
become ACTIVE without a "agent.constitutional_review" AuditEvent in the
ledger that references this agent's manifest version.
