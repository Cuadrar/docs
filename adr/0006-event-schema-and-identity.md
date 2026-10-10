# ADR-0006: Define Event Schema and Identity

- **Status:** Proposed
- **Date:** 2026-10-10
- **Decision Makers:** Cuadrar Maintainers

## Context

Cuadrar collects operational events from heterogeneous sources,
including Kubernetes, applications, messaging systems, and
infrastructure integrations.

These sources expose different event formats, identifiers,
timestamps, and metadata.

Cuadrar requires a unified event schema that allows events
to be processed consistently without losing important
source-specific information.

The schema must support:

- Multiple event sources.
- Stable event identification.
- Event deduplication.
- Cross-source event exploration.
- Service and resource identification.
- Future event correlation.
- Backward-compatible schema evolution.
- Efficient serialization and storage.
- Tenant isolation.
- Extensibility without frequent schema changes.

The schema will be shared between the open-source agent
and Cuadrar Cloud.

It will form part of Cuadrar's public integration contract.

## Decision

Cuadrar will define a versioned Unified Operational Event
schema using JSON as the initial serialization format.

Each event will have a stable UUIDv7 identifier and
structured metadata describing its source, resource,
and associated service.

The canonical schema will be independent of transport
protocols and storage implementations.

### 1. Canonical Event Structure

A Unified Operational Event will contain the following
top-level fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `schema_version` | string | Yes | Event schema version |
| `id` | UUID | Yes | Unique event identifier |
| `timestamp` | string | Yes | Event occurrence timestamp |
| `type` | string | Yes | Canonical event type |
| `severity` | string | Yes | Operational severity |
| `source` | object | Yes | Event origin |
| `resource` | object | No | Associated resource |
| `service` | object | No | Associated service |
| `environment` | string | No | Deployment environment |
| `message` | string | No | Human-readable description |
| `attributes` | object | No | Source-specific metadata |
| `related_event_ids` | array | No | Explicit event relationships |

The initial schema version will be:

```text
1.0
```

### 2. Example Event

```json
{
  "schema_version": "1.0",
  "id": "0199f0b1-7a12-7000-8000-000000000001",
  "timestamp": "2026-10-10T12:00:00Z",
  "type": "kubernetes.pod.restarted",
  "severity": "warning",
  "source": {
    "type": "kubernetes",
    "name": "production-cluster"
  },
  "resource": {
    "type": "kubernetes.pod",
    "id": "7c3e9f42-5e19-4f3c-9b6d-6e1a4c9b0a21",
    "name": "payment-service-7f9d8",
    "namespace": "payments"
  },
  "service": {
    "name": "payment-service"
  },
  "environment": "production",
  "message": "Pod restarted after container failure",
  "attributes": {
    "container": "payment-api",
    "restart_count": 3,
    "reason": "OOMKilled"
  }
}
```

The example represents a normalized Kubernetes
operational event.

Source-specific collectors are responsible for mapping
their native event representations into this schema.

### 3. Event Identity

Every event must have a globally unique identifier.

Cuadrar will use UUIDv7 for event identifiers.

UUIDv7 provides:

- Time-ordered identifier generation.
- Distributed generation without coordination.
- Compatibility with standard UUID storage types.
- Efficient ordering characteristics.
- Low collision probability.

#### Generation

Event identifiers should be generated when an event
enters the normalized event pipeline.

The identifier must remain unchanged during:

- Filtering.
- Enrichment.
- Buffering.
- Batching.
- Delivery retries.
- Cloud ingestion.

An event must not receive a new identifier
during a delivery retry.

#### External Event Identifiers

Some event sources provide their own identifiers.

These identifiers may be preserved in event attributes.

The canonical Cuadrar event identifier remains
independent of source-specific identifiers.

Source integrations may implement additional
deduplication based on native event identity.

The exact source-level deduplication strategy
will be defined separately.

### 4. Event Type Naming

Event types will use lowercase dot-separated identifiers.

Format:

```text
<domain>.<resource>.<action>
```

Examples:

```text
kubernetes.pod.restarted
kubernetes.deployment.updated
kubernetes.node.not_ready

application.request.failed
application.job.failed
application.dependency.unavailable

messaging.consumer.failed
messaging.connection.lost

database.connection.failed
database.instance.unavailable
```

Event types must:

- Use lowercase characters.
- Use dots as namespace separators.
- Avoid whitespace.
- Remain stable across compatible schema versions.
- Describe meaningful operational occurrences.

Custom event types may use application-specific namespaces.

Example:

```text
custom.payment.provider_failed
```

The canonical event type registry will be maintained
separately from this ADR.

### 5. Event Severity

Cuadrar will define five severity levels:

| Severity | Description |
|---|---|
| `debug` | Diagnostic event with minimal operational impact |
| `info` | Normal operational activity |
| `warning` | Potentially problematic condition |
| `error` | Operational failure requiring attention |
| `critical` | Severe failure with significant impact |

Severity must be represented as a lowercase string.

Source integrations must map native severity levels
to the canonical values.

Severity classification may vary between sources.

The MVP will not implement automatic severity
reclassification using AI or anomaly detection.

### 6. Event Timestamp

Every event must contain a timestamp representing
when the observed occurrence happened.

The timestamp must use RFC 3339 format.

Example:

```text
2026-10-10T12:00:00Z
```

All timestamps must be normalized to UTC.

Fractional seconds may be included.

The Cloud ingestion service will additionally record
the time at which the event was received.

Conceptually:

```text
timestamp   = Event occurrence time
received_at = Cloud ingestion time
```

`received_at` is Cloud-managed metadata and is not
part of the agent-provided canonical event schema.

The ingestion service must not replace the original
event timestamp with the ingestion timestamp.

### 7. Source Identity

The `source` object identifies the system or integration
that produced the event.

Structure:

```json
{
  "type": "kubernetes",
  "name": "production-cluster"
}
```

Fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Source integration type |
| `name` | string | No | Human-readable source name |
| `id` | string | No | Stable source identifier |

Source types may include:

- `kubernetes`
- `application`
- `messaging`
- `database`
- `custom`

The source identifier should remain stable across
agent restarts whenever the underlying source
identity remains unchanged.

### 8. Resource Identity

The `resource` object identifies the infrastructure
or application resource associated with the event.

Structure:

```json
{
  "type": "kubernetes.pod",
  "id": "7c3e9f42-5e19-4f3c-9b6d-6e1a4c9b0a21",
  "name": "payment-service-7f9d8",
  "namespace": "payments"
}
```

Fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Resource category |
| `id` | string | No | Stable resource identifier |
| `name` | string | No | Human-readable resource name |
| `namespace` | string | No | Logical resource namespace |

Resource identifiers should use native stable identifiers
when available.

For Kubernetes resources, the Kubernetes object UID
should be preferred over resource names.

Resource names may change or be reused.

The resource identity model must support
non-Kubernetes resources.

### 9. Service Identity

The `service` object identifies the logical application
or service associated with the event.

Structure:

```json
{
  "name": "payment-service",
  "namespace": "payments"
}
```

Fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Logical service name |
| `namespace` | string | No | Service namespace |
| `version` | string | No | Application version |

Service identification may originate from:

- Kubernetes labels.
- Application configuration.
- Collector configuration.
- Source-specific metadata.

The agent should not invent service identities
when reliable information is unavailable.

Cloud-side service resolution and relationship
inference may be introduced separately.

### 10. Environment

The optional `environment` field identifies
the deployment environment.

Examples:

```text
production
staging
development
testing
```

Environment names are user-defined strings.

Cuadrar will not enforce a fixed set
of environment values.

Environment information may be configured
globally or provided by individual event sources.

### 11. Event Attributes

The `attributes` object stores additional
source-specific metadata.

Example:

```json
{
  "container": "payment-api",
  "restart_count": 3,
  "reason": "OOMKilled"
}
```

Attributes must be JSON-compatible.

Supported value types include:

- String.
- Number.
- Boolean.
- Array.
- Object.
- Null.

The ingestion API will enforce limits on:

- Attribute count.
- Attribute key length.
- Attribute value size.
- Maximum nesting depth.
- Total event size.

These limits will be defined in the
public ingestion API specification.

Attributes must not implicitly contain
secrets or unrestricted application payloads.

### 12. Event Relationships

The schema will support optional explicit
relationships between events.

Field:

```json
{
  "related_event_ids": [
    "0199f0b1-7a12-7000-8000-000000000002"
  ]
}
```

This field represents relationships explicitly
known by the event producer or collector.

It must not be used for speculative relationships.

Future Cloud correlation capabilities may
infer additional relationships independently.

The MVP does not require automated event correlation.

### 13. Event Immutability

Operational events will be immutable after ingestion.

Once an event has been successfully persisted:

- Its identifier must not change.
- Its timestamp must not change.
- Its canonical attributes must not be modified.
- Duplicate submissions must not overwrite existing data.

Cloud-generated metadata and derived relationships
may be stored separately.

This separation allows future analytical processing
without modifying original events.

### 14. Tenant Identity

Tenant identity will not be included in the
canonical agent-provided event schema.

The Cloud ingestion service will determine:

- Organization identity.
- Project identity.

These values will be derived from authenticated
project-scoped credentials.

The agent must not be allowed to select
an arbitrary tenant through event payload fields.

Tenant metadata will be associated with events
during Cloud ingestion.

### 15. Event Deduplication

Cloud deduplication will use:

```text
(project_id, event_id)
```

This combination uniquely identifies an event
within a Cuadrar project.

Duplicate submissions must not create
additional stored events.

The deduplication strategy follows ADR-0005.

#### Source-Level Deduplication

Source collectors may receive the same native
event multiple times.

Where reliable source identifiers exist,
collectors should avoid generating multiple
canonical events for the same observed occurrence.

Source-level deduplication is separate from
Cloud ingestion deduplication.

The MVP does not guarantee elimination
of all semantically duplicate source events.

### 16. Schema Versioning

Every event will contain a `schema_version` field.

The initial version will be:

```text
1.0
```

Schema versions will use a major.minor format.

#### Minor Versions

Minor versions may introduce backward-compatible
changes, such as:

- New optional fields.
- New supported attribute structures.
- Additional event categories.

Existing required fields must not be removed
within a minor version.

#### Major Versions

Major versions may introduce breaking changes.

Examples include:

- Removing required fields.
- Changing field types.
- Changing identity semantics.
- Restructuring core event objects.

Breaking changes must be documented and
introduced through an explicit migration strategy.

Cuadrar Cloud must support compatible agent
versions without requiring simultaneous upgrades.

### 17. Schema Validation

The canonical event schema will be published
as a machine-readable JSON Schema.

The schema will be maintained in the
public documentation repository.

Proposed location:

```text
schemas/events/v1.0.schema.json
```

Both the agent and Cloud should validate
events against the same documented contract.

Implementations may use generated types
or equivalent language-specific validation.

The public schema will serve as the
source of truth for event compatibility.

### 18. Future Extensibility

The event model should support future additions
such as:

- Distributed trace references.
- Correlation identifiers.
- Deployment identifiers.
- Infrastructure topology metadata.
- Event relationship types.
- Incident references.
- Additional source categories.

Future extensions must preserve compatibility
with supported schema versions.

The canonical event model must not depend
on a particular transport protocol.

## Alternatives Considered

### 1. Unstructured JSON Events

Events are stored as arbitrary JSON objects
without a canonical schema.

**Advantages:**

- Maximum flexibility.
- Minimal initial normalization.
- Easy integration with heterogeneous sources.

**Disadvantages:**

- Inconsistent event semantics.
- Difficult cross-source querying.
- Limited validation.
- Complicated relationship analysis.
- Poor long-term compatibility guarantees.

**Reason for Rejection:**

Cuadrar requires consistent event semantics
to support cross-source exploration and analysis.

### 2. Flat Event Schema

All event attributes are represented as
top-level scalar fields.

**Advantages:**

- Simple serialization.
- Straightforward database mapping.
- Easy basic filtering.

**Disadvantages:**

- Limited representation of complex identities.
- Difficult extension.
- Increased naming conflicts.
- Poor support for heterogeneous resources.

**Reason for Rejection:**

Structured source, resource, and service identities
provide greater flexibility for future analysis.

### 3. OpenTelemetry Native Data Model

Cuadrar adopts OpenTelemetry signal formats
as its canonical event representation.

**Advantages:**

- Established ecosystem.
- Existing SDKs and collectors.
- Mature protocol definitions.
- Broad language support.

**Disadvantages:**

- Operational events do not map directly
  to a single OpenTelemetry signal type.
- Additional complexity for event-specific semantics.
- Greater dependency on external schema evolution.
- Potential mismatch with Cuadrar's product model.

**Reason for Rejection:**

Cuadrar requires an event-oriented canonical model
optimized for operational occurrences and relationships.

OpenTelemetry compatibility may be implemented
through dedicated receivers and adapters.

### 4. Unified Operational Event Schema

Cuadrar defines a versioned canonical event model
with structured identities and extensible attributes.

**Advantages:**

- Consistent event semantics.
- Stable public contract.
- Support for heterogeneous sources.
- Explicit identity and deduplication.
- Foundation for future relationship analysis.

**Disadvantages:**

- Requires normalization logic.
- Introduces schema governance responsibilities.
- Source-specific information may require mapping.
- Schema evolution requires compatibility testing.

**Reason for Selection:**

This approach best supports Cuadrar's event-oriented
architecture and long-term product direction.

## Consequences

### Positive

- Consistent event representation across integrations.
- Stable event identifiers.
- Reliable Cloud-side deduplication.
- Clear separation between event identity and tenant identity.
- Support for structured resource and service metadata.
- Extensible source-specific attributes.
- Transport-independent event contracts.
- Foundation for future event correlation.
- Explicit schema compatibility rules.

### Negative

- Collectors must implement normalization.
- Schema changes require compatibility management.
- Structured identities increase payload size.
- Some native source metadata may not map directly.
- UUIDv7 generation introduces a dependency
  on correct identifier generation.
- Cross-source identity consistency remains challenging.
- Source-level semantic duplicates may still occur.

## MVP Implications

The MVP will implement:

- Unified Operational Event schema version 1.0.
- UUIDv7 event identifiers.
- RFC 3339 UTC timestamps.
- Canonical event type naming.
- Five severity levels.
- Structured source identity.
- Structured resource identity.
- Optional service identity.
- Optional environment.
- Extensible JSON attributes.
- Event immutability.
- Project-scoped event deduplication.
- JSON Schema validation.
- Public schema documentation.

The MVP will not implement:

- Automated event correlation.
- AI-based event classification.
- Automatic service topology discovery.
- Distributed trace correlation.
- Cross-project event relationships.
- Complex semantic event deduplication.
- Automatic event schema migration.

## Related Decisions

- ADR-0001: Adopt Unified Operational Events as the Core Data Model
- ADR-0002: Define Open-Source and Cloud Responsibilities
- ADR-0003: Define Event Transport and Delivery Semantics
- ADR-0004: Define Agent Architecture
- ADR-0005: Define Cloud Ingestion Architecture
- ADR-0007: Define Multi-Tenancy and Authentication (Planned)
