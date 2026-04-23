# Aevum Protocol Specification
## Section 03: Non-Goals

This section is normative. The boundaries defined here are enforced by the
RFC process defined in the Aevum governance documents. Removing or narrowing
a non-goal requires a formal RFC and unanimous maintainer approval.

---

An implementation that provides any of the following as part of its core
surface is not an Aevum implementation, even if it also implements the
five functions.

### 3.1 Not a Data Integration Platform

Aevum MUST NOT provide pipeline management, schema transformation, or
data movement between external systems. Complications MAY ingest data
from external sources, but the kernel does not manage pipelines.
Comparable tools: Airbyte, Fivetran, dbt, MuleSoft.

### 3.2 Not an AI Orchestration Framework

Aevum MUST NOT chain prompts, manage agent loops, or schedule agent
execution. The `query` function assembles context for AI consumers;
it does not orchestrate them.
Comparable tools: LangChain, LlamaIndex, AutoGen, CrewAI.

### 3.3 Not a Compliance Report Generator

Aevum produces an episodic ledger that can serve as evidence for
compliance audits. It MUST NOT interpret regulations, generate compliance
reports, or provide legal conclusions. The ledger is a technical artifact.

### 3.4 Not a Knowledge Graph Database

Aevum uses a graph internally to represent relationships. It MUST NOT
expose a Cypher, Gremlin, or SPARQL endpoint to end users. Graph
traversal is an implementation detail of the `query` function.
Comparable tools: Neo4j, Amazon Neptune, Stardog.

### 3.5 Not an Agent Execution Environment

Aevum governs agents that operate through its membrane. It MUST NOT
schedule, run, or host agent processes. Agents are external; they call
the five functions like any other consumer.

### 3.6 Not a Streaming Message Broker

Aevum appends events to the episodic ledger. It MUST NOT stream events
to consumers in real time or replace a message broker.
Comparable tools: Apache Kafka, Apache Pulsar, NATS.

### 3.7 Not an Observability Backend

Aevum emits OpenTelemetry spans and metrics. It MUST NOT store or
visualize observability data.
Comparable tools: Datadog, Grafana, Honeycomb.

### 3.8 Not an Identity Provider

Aevum resolves identity through the OIDC complication at query time.
It MUST NOT store credentials, issue tokens, or replace an external IDP.

### 3.9 Not a General-Purpose Policy Engine

OPA and Cedar are embedded to govern Aevum's own operations. They MUST NOT
be exposed as a general policy evaluation service for other applications.
Comparable tools: OPA standalone, AWS Cedar.
