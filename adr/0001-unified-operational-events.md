# ADR-0001: Adopt Unified Operational Events as the Core Data Model

**Status:** Proposed  
**Date:** 2026-10-10  
**Decision Makers:** Cuadrar Maintainers

## Context

Modern distributed systems generate operational information across multiple layers, including applications, infrastructure, messaging systems, and databases.

This information is typically fragmented across different observability tools, making it difficult for engineers to understand relationships between operational events and their impact on system behavior.

Traditional observability platforms primarily focus on metrics, logs, and traces. While these signals provide valuable information, understanding how individual operational changes and failures relate to each other often requires manual investigation.

Cuadrar aims to provide a unified view of operational events across distributed systems.

To support this goal, Cuadrar requires a consistent event model that can represent different types of operational activity without being tied to a specific infrastructure platform, application framework, or event source.

## Decision

Cuadrar will adopt **Unified Operational Events** as its core data model.

An operational event represents a meaningful occurrence, state transition, or detected condition within an observed system.

Events may originate from different layers, including:

- **Infrastructure:** Pod restarts, deployment changes, scaling events, and node failures.
- **Applications:** Request failures, background job failures, and application lifecycle events.
- **Messaging:** Consumer failures, connection interruptions, and processing errors.
- **Databases:** Connection failures, availability changes, and operational errors.
- **Custom Sources:** User-defined operational events emitted through supported integrations.

All supported sources will be normalized into a common event representation before being processed by Cuadrar Cloud.

### Core Event Attributes

The unified event model will support the following conceptual attributes:

| Attribute | Description |
|---|---|
| `id` | Unique identifier of the event |
| `timestamp` | Time at which the event occurred |
| `source` | System or integration that produced the event |
| `type` | Event type identifying the occurrence |
| `severity` | Classification of the event's operational significance |
| `service` | Associated application or service, when available |
| `environment` | Environment in which the event occurred, when available |
| `resource` | Infrastructure or application resource associated with the event |
| `attributes` | Additional source-specific event metadata |

The exact serialization format, field types, required attributes, and versioning strategy will be defined separately.

### Event Collection

Cuadrar will follow an explicit observation model.

Only events enabled through agent configuration or supported integration settings should be collected and forwarded.

The agent must not indiscriminately forward every event available from an observed system.

Event collection and normalization should be designed to minimize resource consumption and unnecessary network traffic.

### Event Processing

Cuadrar Cloud will consume normalized operational events and provide capabilities for:

- Event ingestion and validation.
- Event storage and retrieval.
- Filtering and exploration.
- Associating events with services and resources.
- Identifying relationships between operational events.

Advanced correlation and automated root cause analysis are not required for the initial MVP.

### Extensibility

The unified event model must support additional event sources without requiring fundamental changes to the core architecture.

Source-specific information may be preserved through extensible event attributes.

Cuadrar must not assume that all observed systems run on Kubernetes.

## Alternatives Considered

### 1. Infrastructure-Only Events

Focus exclusively on infrastructure events, primarily Kubernetes.

**Advantages:**
- Smaller initial integration scope.
- Simpler event collection.
- Faster initial implementation.

**Disadvantages:**
- Limited visibility into application behavior.
- Strong dependency on infrastructure-specific concepts.
- Reduced ability to understand cross-layer operational relationships.

**Reason for Rejection:** This approach would unnecessarily restrict Cuadrar's long-term product capabilities.

### 2. Application-Only Events

Focus on application-level failures and operational activities.

**Advantages:**
- Direct relevance to backend engineers.
- Strong alignment with application debugging workflows.

**Disadvantages:**
- Limited visibility into infrastructure changes.
- Additional integration requirements.
- Incomplete operational context.

**Reason for Rejection:** Application events alone cannot provide sufficient context for understanding distributed system behavior.

### 3. Unified Operational Events

Represent infrastructure, application, and integration events through a common model.

**Advantages:**
- Consistent representation across different sources.
- Extensible architecture.
- Support for cross-layer event relationships.
- Independence from specific infrastructure technologies.

**Disadvantages:**
- Additional normalization complexity.
- Need for carefully designed event semantics.
- Potential inconsistencies between source-specific event attributes.

**Reason for Selection:** This approach best supports Cuadrar's long-term product direction while allowing the initial implementation to remain limited in scope.

## Consequences

### Positive

- A consistent event representation across supported integrations.
- New event sources can be introduced incrementally.
- Cloud processing can remain largely independent of individual event sources.
- Events from different system layers can be explored together.
- The architecture provides a foundation for future event correlation and incident investigation capabilities.

### Negative

- Event normalization introduces additional implementation complexity.
- Source-specific details may not map directly to common attributes.
- Event schema evolution will require compatibility management.
- Relationships between events may be incomplete when source metadata is insufficient.

## MVP Implications

The MVP will implement a limited set of event sources using the unified event model.

Initial integrations should prioritize Kubernetes operational events and a small set of application-level events.

The MVP does not require support for every event category described in this ADR.

Advanced event correlation, automated root cause analysis, and AI-powered investigation are explicitly outside the scope of this decision.

The exact initial integrations and supported event types will be defined in the MVP Product Scope.

## Related Decisions

- ADR-0002: Define Open-Source and Cloud Responsibilities (Planned)
- ADR-0003: Define Event Transport and Delivery Semantics (Planned)
- ADR-0004: Define Agent Architecture (Planned)
