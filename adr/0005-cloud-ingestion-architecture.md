# ADR-0005: Define Cloud Ingestion Architecture

- **Status:** Proposed
- **Date:** 2026-10-10
- **Decision Makers:** Cuadrar Maintainers

## Context

Cuadrar Cloud is responsible for receiving, validating, storing,
and exposing operational events collected by Cuadrar Agents.

The ingestion architecture must support the Unified Operational
Event model defined in ADR-0001 and the delivery semantics
established in ADR-0003.

The initial implementation must prioritize:

- Low infrastructure and operational costs.
- Reliable event ingestion.
- Idempotent event processing.
- Secure multi-tenant isolation.
- Simple deployment and maintenance.
- Predictable resource consumption.
- Efficient event retrieval.
- Independent scaling of future components.
- Compatibility with multiple agent versions.

The MVP is expected to operate with relatively low event volumes.

The architecture must avoid introducing distributed infrastructure
components before they are justified by actual workload requirements.

At the same time, the design must provide clear boundaries
for introducing dedicated ingestion workers, message brokers,
and analytical databases in future releases.

## Decision

Cuadrar Cloud will initially use a Go-based modular monolith
with PostgreSQL as its primary persistent storage.

Event ingestion will use synchronous database persistence
before acknowledging successful delivery.

The MVP will not require a message broker or separate
analytical database.

### 1. High-Level Architecture

The initial architecture will consist of:

- HTTP API.
- Authentication and authorization.
- Event validation.
- Event ingestion service.
- PostgreSQL database.
- Event query service.
- Organization and project management.
- Web application.

Conceptual architecture:

```text
                  Cuadrar Agents
                        |
                        | HTTPS / JSON
                        v
              +--------------------+
              | Cloudflare         |
              | DNS / Proxy        |
              +--------------------+
                        |
                        v
              +--------------------+
              | Cuadrar Cloud API  |
              |                    |
              | Authentication     |
              | Rate Limiting      |
              | Request Validation |
              +--------------------+
                        |
                        v
              +--------------------+
              | Ingestion Service  |
              |                    |
              | Event Validation   |
              | Tenant Resolution  |
              | Deduplication      |
              | Persistence        |
              +--------------------+
                        |
                        v
              +--------------------+
              | PostgreSQL         |
              |                    |
              | Organizations      |
              | Projects           |
              | Credentials        |
              | Events             |
              +--------------------+
                        ^
                        |
              +--------------------+
              | Event Query API    |
              +--------------------+
                        ^
                        |
              +--------------------+
              | Cuadrar Web        |
              +--------------------+
```

The API and ingestion service will initially run within
the same application process.

Logical component boundaries must remain explicit.

Cloudflare is an optional infrastructure entry point,
not a dependency of the ingestion domain logic.

### 2. Backend Architecture

Cuadrar Cloud will be implemented in Go.

The initial architecture will follow a modular monolith approach.

The application will contain separate logical modules for:

- Organizations.
- Projects.
- Agent credentials.
- Event ingestion.
- Event storage.
- Event querying.
- Usage accounting.

Modules should communicate through internal interfaces
rather than network-based service calls.

The architecture must not require independently deployed
microservices for the MVP.

#### Design Principles

- Business logic must remain independent of HTTP handlers.
- Database access must be isolated behind storage interfaces.
- Event validation must be independent of persistence.
- Authentication must be separated from ingestion logic.
- Components must support independent unit testing.
- Database transactions must have explicit boundaries.

The initial implementation may use a single executable
containing both ingestion and query APIs.

### 3. Event Ingestion API

The ingestion API will expose:

```http
POST /v1/events
```

The endpoint will accept batches of normalized operational events.

Requests must use HTTPS.

Supported request formats:

- JSON.
- gzip-compressed JSON.

The API will enforce:

- Authentication.
- Maximum request size.
- Maximum decompressed payload size.
- Maximum event count per batch.
- Event schema validation.
- Project-level access control.
- Rate limits.

The exact request and response schemas will be defined
in the public ingestion API specification.

#### Request Processing

The ingestion request lifecycle will be:

1. Receive the HTTP request.
2. Authenticate the API key.
3. Resolve the associated organization and project.
4. Enforce request limits.
5. Decode the request body.
6. Validate the event batch.
7. Associate events with the authenticated project.
8. Persist events using a database transaction.
9. Return a successful acknowledgement.

The API must not trust organization or project identifiers
provided by the agent.

Tenant identity must be derived from authenticated credentials.

### 4. Event Validation

The ingestion service will validate incoming events
before persistence.

Validation will include:

- Required event fields.
- Supported event schema version.
- Event identifier format.
- Timestamp format.
- Maximum attribute sizes.
- Maximum event payload size.
- Supported field types.

Invalid events must not be persisted.

The MVP will use atomic batch validation.

If any event in a batch is invalid, the entire batch
will be rejected.

The API must return an appropriate validation error.

Partial batch acceptance is not supported.

### 5. Event Persistence

PostgreSQL will be the primary event storage engine
for the MVP.

The ingestion service will persist accepted events
within database transactions.

A successful HTTP acknowledgement must only be returned
after the transaction has committed successfully.

The transaction boundary will include all event inserts
required for the batch.

The system must not acknowledge events that exist
only in application memory.

#### Persistence Guarantees

The ingestion service will provide:

- Atomic batch persistence.
- Transactional consistency.
- Duplicate event protection.
- Tenant-scoped event storage.

A database failure must result in an unsuccessful
ingestion response.

The agent may retry the request according to ADR-0003.

### 6. Event Deduplication

Cuadrar Cloud must support idempotent event ingestion.

Each operational event will contain a stable identifier.

The combination of project identifier and event identifier
will define event uniqueness.

Conceptual uniqueness constraint:

```sql
UNIQUE (project_id, event_id)
```

The ingestion service must handle duplicate submissions
without creating additional event records.

#### Duplicate Handling

When a previously accepted event is submitted again:

- The existing event must not be modified.
- No additional event record should be created.
- The duplicate must be treated as successfully accepted.
- The response must remain compatible with agent retries.

For the MVP, duplicate detection will rely on PostgreSQL
uniqueness enforcement.

The implementation may use:

```sql
INSERT INTO events (...)
VALUES (...)
ON CONFLICT DO NOTHING;
```

The actual implementation must use the appropriate
uniqueness constraint.

Duplicate event identifiers within the same incoming batch
must also be handled consistently.

A repeated event identifier with different event content
will be treated as a duplicate rather than an update.

The deduplication guarantee applies only while the
corresponding uniqueness record is retained.

Retention-related deduplication limitations must be documented.

### 7. PostgreSQL Storage Model

PostgreSQL will store both operational events
and application metadata.

The logical data model will include:

```text
organizations
    |
    +-- projects
           |
           +-- agent_credentials
           |
           +-- events
           |
           +-- usage_records
```

#### Event Storage

The event storage model must support:

- Project isolation.
- Time-range queries.
- Event type filtering.
- Severity filtering.
- Service filtering.
- Resource filtering.
- Event identifier lookup.

A conceptual event table:

```sql
CREATE TABLE events (
    project_id UUID NOT NULL,
    event_id UUID NOT NULL,

    occurred_at TIMESTAMPTZ NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    source TEXT NOT NULL,
    event_type TEXT NOT NULL,
    severity TEXT,

    service TEXT,
    environment TEXT,
    resource TEXT,

    attributes JSONB NOT NULL DEFAULT '{}',

    PRIMARY KEY (project_id, event_id)
);
```

This schema is illustrative.

The final schema will be defined separately.

#### Indexing

Indexes should prioritize common access patterns:

- Project and time range.
- Project and event type.
- Project and service.
- Project and severity.

Indexes must be introduced based on actual query patterns.

Unnecessary indexes should be avoided because they
increase write overhead and storage consumption.

#### JSONB Attributes

Source-specific event attributes will be stored using JSONB.

This allows different event sources to preserve additional
metadata without requiring schema changes for every
new integration.

Frequently queried attributes may later be promoted
to dedicated columns or indexed expressions.

The MVP will not support arbitrary high-performance
search across every JSONB attribute.

### 8. Multi-Tenancy

Cuadrar Cloud will use a shared database with logical
tenant isolation.

The hierarchy will be:

```text
Organization
    |
    +-- Project
           |
           +-- Agent Credentials
           |
           +-- Operational Events
```

Each event will belong to exactly one project.

All event queries must be scoped to an authorized project.

Tenant isolation must be enforced at the application
and data-access boundaries.

The system must not rely exclusively on frontend filtering
to enforce tenant isolation.

Database-level Row-Level Security may be introduced
as an additional defense mechanism.

The exact authorization model will be defined separately.

### 9. Event Querying

The Cloud API will expose event retrieval capabilities
for the web application.

The initial query interface should support:

- Project selection.
- Time-range filtering.
- Event type filtering.
- Severity filtering.
- Service filtering.
- Resource filtering.
- Pagination.
- Reverse chronological ordering.

The query service must enforce tenant authorization.

The API must apply limits to prevent expensive
unbounded database queries.

Cursor-based pagination is preferred for large event sets.

The exact API contract will be defined separately.

### 10. Event Retention

Cuadrar Cloud will enforce configurable event retention.

Retention periods may depend on subscription plans.

Expired events must be removed from active storage.

The MVP may use scheduled database cleanup operations.

The cleanup process must:

- Operate in bounded batches.
- Avoid unnecessarily long transactions.
- Minimize interference with ingestion.
- Expose operational metrics.

Time-based partitioning may be introduced when event
volume makes row-level cleanup inefficient.

Retention policies must account for their impact
on event deduplication guarantees.

The exact retention periods will be defined
during product and pricing design.

### 11. Rate Limiting and Usage Protection

The ingestion API must protect Cloud infrastructure
from excessive traffic.

The MVP will enforce:

- Maximum request body size.
- Maximum events per batch.
- Maximum event attribute size.
- Request timeouts.
- Database connection limits.
- Project-level ingestion limits.

Rate limiting may be implemented at the application
or infrastructure layer.

Cloudflare may provide additional edge-level protection.

Application-level limits remain necessary because
edge-level controls do not replace tenant-aware
usage enforcement.

When a project exceeds its allowed ingestion rate,
the API should return:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

Agents must handle this response according to ADR-0003.

### 12. Authentication

Agent ingestion will use project-scoped API keys.

API keys must:

- Be generated using a cryptographically secure mechanism.
- Be associated with a specific project.
- Support revocation.
- Be stored securely.
- Never be logged in plaintext.
- Be transmitted only over HTTPS.

The ingestion service must resolve project identity
from the authenticated API key.

API key validation should avoid unnecessary database
work where practical.

Credential caching may be introduced if required
by ingestion performance.

The exact credential format and lifecycle will be
defined separately.

### 13. Scaling Strategy

The MVP will initially run with:

- One Cloud API instance.
- One PostgreSQL database.
- No external message broker.
- No dedicated ingestion workers.

The application should remain stateless with respect
to accepted event storage.

This allows additional API instances to be introduced
without changing the ingestion contract.

#### Horizontal Scaling

Future scaling may include:

- Multiple API instances.
- Dedicated ingestion workers.
- Database connection pooling.
- Read replicas.
- Event storage partitioning.
- Separate analytical storage.

The MVP must not depend on in-memory application state
for event persistence or deduplication.

### 14. Future Asynchronous Ingestion

The architecture must allow the introduction of
asynchronous ingestion without changing the public API.

A possible future architecture:

```text
Cuadrar Agents
      |
      v
+------------------+
| Ingestion API    |
+------------------+
      |
      v
+------------------+
| Durable Queue    |
| NATS / Kafka     |
+------------------+
      |
      v
+------------------+
| Ingestion Workers|
+------------------+
      |
      +------------------+
      |                  |
      v                  v
+-------------+   +-------------+
| PostgreSQL  |   | ClickHouse  |
| Metadata    |   | Events      |
+-------------+   +-------------+
```

In this architecture, the reliable ingestion boundary
may move from PostgreSQL transaction commit to durable
message broker acknowledgement.

Such a change must preserve the delivery guarantees
defined in ADR-0003.

The MVP does not require this architecture.

### 15. Cloud Observability

The Cloud application must expose operational metrics.

At minimum:

| Metric | Description |
|---|---|
| `cuadrar_ingestion_requests_total` | Total ingestion requests |
| `cuadrar_ingestion_events_total` | Total newly persisted events |
| `cuadrar_ingestion_duplicates_total` | Total duplicate events |
| `cuadrar_ingestion_rejected_total` | Total rejected events |
| `cuadrar_ingestion_duration_seconds` | Ingestion request duration |
| `cuadrar_ingestion_batch_size` | Number of events per batch |
| `cuadrar_database_errors_total` | Total database operation failures |

The application must also provide:

- Structured logging.
- Request identifiers.
- Database health monitoring.
- Error reporting.
- Basic readiness and liveness endpoints.

Sensitive event attributes and authentication credentials
must not be included in operational logs by default.

### 16. Failure Handling

#### Database Unavailability

When PostgreSQL is unavailable:

- New event batches must not be acknowledged.
- The API should return a retryable error.
- Agents may retry according to ADR-0003.
- No fallback to unbounded in-memory storage is permitted.

#### Request Timeout

If a request times out after the database transaction
has committed, the agent may retry the batch.

Deduplication must prevent duplicate persistence.

#### Application Restart

An application restart must not affect previously
committed events.

In-flight requests may fail and be retried by agents.

#### Invalid Event Batch

Invalid batches must be rejected without partial
persistence.

Validation errors should provide enough information
for troubleshooting without exposing sensitive data.

### 17. Security

The ingestion architecture must follow secure-by-default
principles.

Requirements include:

- HTTPS for external communication.
- Project-scoped authentication.
- Tenant authorization.
- Request size limits.
- Safe JSON decoding.
- Parameterized database queries.
- Secure credential storage.
- Protection against decompression bombs.
- Restricted database permissions.
- Structured security-related logging.

The system must not expose internal database
connections directly to agents.

### 18. Testing Strategy

The ingestion implementation must include:

- Unit tests for event validation.
- Authentication tests.
- Tenant isolation tests.
- Duplicate ingestion tests.
- Concurrent duplicate submission tests.
- Atomic batch persistence tests.
- Database failure tests.
- Request timeout tests.
- Rate limiting tests.
- Compressed payload tests.
- Integration tests using PostgreSQL.
- End-to-end tests with Cuadrar Agent.

Performance tests should measure:

- Events ingested per second.
- Ingestion latency.
- Database write throughput.
- Query latency.
- Storage growth.
- CPU and memory consumption.

Performance targets will be established
during MVP validation.

## Alternatives Considered

### 1. PostgreSQL with Synchronous Ingestion

Events are persisted directly to PostgreSQL
before successful acknowledgement.

**Advantages:**

- Minimal infrastructure.
- Strong transactional guarantees.
- Simple deduplication.
- Low operational complexity.
- Suitable for early-stage workloads.

**Disadvantages:**

- Ingestion throughput depends on database performance.
- Analytical queries compete with writes.
- PostgreSQL storage may become expensive at scale.

**Reason for Selection:**

This approach provides the simplest reliable
architecture for the initial MVP.

### 2. PostgreSQL with Message Broker

Events are first written to a durable message broker
and later persisted asynchronously.

**Advantages:**

- Decoupled ingestion and persistence.
- Better burst handling.
- Independent worker scaling.
- Improved resilience during storage outages.

**Disadvantages:**

- Additional infrastructure costs.
- More complicated operations.
- Increased implementation complexity.
- Additional delivery and consistency considerations.

**Reason for Rejection:**

The expected MVP workload does not justify
a dedicated message broker.

### 3. ClickHouse as Primary Event Storage

Operational events are stored directly in ClickHouse.

**Advantages:**

- Efficient analytical queries.
- High ingestion throughput.
- Column-oriented compression.
- Strong performance for large event datasets.

**Disadvantages:**

- Additional operational complexity.
- Different transaction and deduplication semantics.
- Less suitable for application metadata.
- More complex small-scale deployment.

**Reason for Rejection:**

The MVP does not require ClickHouse-scale analytics.

ClickHouse remains a potential future event storage engine.

### 4. Fully Serverless Ingestion

Events are ingested through serverless functions
and managed storage services.

**Advantages:**

- Minimal server management.
- Automatic scaling.
- Potentially low costs at very small volumes.

**Disadvantages:**

- Platform-specific constraints.
- Potentially unpredictable costs.
- Execution and connection management limitations.
- Increased vendor dependency.

**Reason for Rejection:**

A containerized Go application provides greater
deployment flexibility and predictable behavior.

## Consequences

### Positive

- Low initial infrastructure complexity.
- Minimal number of operational dependencies.
- Reliable transactional event persistence.
- Straightforward event deduplication.
- Simple local development environment.
- Clear tenant isolation boundaries.
- Easy integration with the existing Go ecosystem.
- A gradual migration path toward distributed ingestion.

### Negative

- PostgreSQL may become a performance bottleneck.
- Analytical queries share resources with ingestion.
- Database availability directly affects ingestion.
- Large event volumes may require storage migration.
- Synchronous persistence increases ingestion latency.
- Retention cleanup requires careful implementation.
- The initial architecture has limited independent
  scaling capabilities.

## MVP Implications

The MVP will implement:

- Go-based Cloud backend.
- Modular monolith architecture.
- HTTPS ingestion API.
- JSON and gzip support.
- Project-scoped API key authentication.
- Event validation.
- Atomic batch persistence.
- PostgreSQL event storage.
- PostgreSQL-based deduplication.
- Logical multi-tenant isolation.
- Basic event querying.
- Event retention cleanup.
- Rate limiting.
- Ingestion metrics.
- Structured logging.
- Container-based deployment.

The MVP will not implement:

- Kafka.
- NATS JetStream.
- ClickHouse.
- Dedicated ingestion workers.
- Distributed event processing.
- Multi-region ingestion.
- Event streaming subscriptions.
- Complex analytical query engines.
- Long-term archival storage.

## Related Decisions

- ADR-0001: Adopt Unified Operational Events as the Core Data Model
- ADR-0002: Define Open-Source and Cloud Responsibilities
- ADR-0003: Define Event Transport and Delivery Semantics
- ADR-0004: Define Agent Architecture
- ADR-0006: Define Event Schema and Identity (Planned)
- ADR-0007: Define Multi-Tenancy and Authentication (Planned)
