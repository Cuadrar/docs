# ADR-0003: Define Event Transport and Delivery Semantics

- **Status:** Proposed
- **Date:** 2026-10-10
- **Decision Makers:** Cuadrar Maintainers

## Context

Cuadrar Agent collects operational events from distributed systems
and delivers them to configured destinations.

The primary destination for the MVP is Cuadrar Cloud.

Operational events may be generated at different rates depending
on the observed environment.

Event delivery must account for:

- Temporary network failures.
- Cloud ingestion unavailability.
- High event volumes.
- Limited agent resources.
- Duplicate event delivery.
- Agent restarts and unexpected termination.
- Different agent and cloud release cycles.

The transport mechanism should remain simple enough for the MVP
while providing a foundation for future scalability.

## Decision

Cuadrar will use asynchronous, batched event delivery over HTTPS.

The initial implementation will use a JSON-based HTTP API.

The agent will maintain a bounded in-memory event queue and
deliver events asynchronously through configurable outputs.

### 1. Transport Protocol

The primary transport protocol will be HTTPS.

The agent will communicate with Cuadrar Cloud using an HTTP API.

The initial ingestion endpoint will be:

```http
POST /v1/events
```

Requests must use TLS.

The Cloud API will accept batches of normalized operational events.

Example request:

```http
POST /v1/events HTTP/1.1
Host: api.cuadrar.dev
Authorization: Bearer <api-key>
Content-Type: application/json
Content-Encoding: gzip
```

The exact API request and response schemas will be defined
in the ingestion API specification.

### 2. Event Serialization

Events will be serialized using JSON.

The transport payload will contain an array of normalized
operational events.

Example:

```json
{
  "events": [
    {
      "id": "0199f0b1-7a12-7000-8000-000000000001",
      "timestamp": "2026-10-10T12:00:00Z",
      "source": "kubernetes",
      "type": "kubernetes.pod.restarted",
      "severity": "warning",
      "service": "payment-service",
      "environment": "production",
      "resource": "pod/payment-service-7f9d8",
      "attributes": {
        "namespace": "payments",
        "restart_count": 3
      }
    }
  ]
}
```

This payload is illustrative.

The canonical event schema, identifier format, and required
fields will be defined separately.

### 3. Event Batching

The agent will batch events before sending them to Cloud.

Batching reduces network overhead and improves ingestion
efficiency.

A batch will be flushed when:

- The maximum number of events is reached.
- The configured flush interval expires.
- The configured maximum batch size is reached.

Initial default values:

| Parameter | Default |
|---|---|
| Maximum events per batch | 100 |
| Flush interval | 5 seconds |
| Maximum uncompressed batch size | 1 MiB |
| Request timeout | 10 seconds |

These values are initial implementation defaults and may
be adjusted based on performance testing.

### 4. Event Buffering

The agent will maintain a bounded in-memory event queue.

The queue will temporarily hold normalized events waiting
for delivery.

Initial default configuration:

| Parameter | Default |
|---|---|
| Maximum buffered events | 10,000 |
| Buffer implementation | In-memory |
| Persistent storage | Disabled |

When the queue reaches its configured capacity, the agent
will apply an explicit overflow policy.

The default overflow policy will be:

**Drop newest events.**

This prevents unbounded memory growth and avoids blocking
event collection indefinitely.

Dropped events must be reflected in agent metrics.

The agent must not silently discard events without
operational visibility.

### 5. Delivery Guarantees

Cuadrar will use an at-least-once delivery strategy for
events retained in the agent's active delivery buffer.

Events will remain eligible for retry until:

- The destination acknowledges successful delivery.
- The configured retry limits are exceeded.
- The agent terminates.
- The event is explicitly discarded due to resource limits.

The MVP does not provide durable end-to-end delivery guarantees.

Events may be lost during:

- Agent crashes.
- Agent restarts.
- Buffer overflow.
- Prolonged destination unavailability.
- Exhaustion of configured retry limits.

Exactly-once delivery is not guaranteed.

### 6. Retry Strategy

The agent will retry transient delivery failures using
exponential backoff with jitter.

Initial retry configuration:

| Parameter | Default |
|---|---|
| Initial retry delay | 1 second |
| Maximum retry delay | 30 seconds |
| Backoff strategy | Exponential with jitter |
| Maximum retry attempts | Configurable |
| Default retry behavior | Retry while events remain buffered |

Retryable failures include:

- Network connection failures.
- Request timeouts.
- HTTP 429 responses.
- HTTP 5xx responses.

Non-retryable failures include:

- HTTP 400 Bad Request.
- HTTP 401 Unauthorized.
- HTTP 403 Forbidden.
- Other permanent validation failures.

HTTP 413 Payload Too Large will require special handling
through batch splitting or size enforcement.

When the Cloud API provides a valid `Retry-After` header,
the agent should respect it.

Retries must reuse the original event identifiers.

### 7. Cloud Acknowledgement

Cuadrar Cloud will acknowledge successfully accepted batches
using an HTTP success response.

The MVP will use atomic batch acceptance.

A batch is either:

- Fully accepted.
- Fully rejected.

Partial batch acceptance is not supported in the MVP.

A successful acknowledgement indicates that Cloud has
accepted responsibility for the events.

The Cloud ingestion implementation must not acknowledge
a batch before it has reached the configured reliable
ingestion boundary.

The exact internal ingestion and persistence mechanism
will be defined separately.

### 8. Event Deduplication

Because the delivery strategy allows retries, duplicate
events may reach Cuadrar Cloud.

Each event will contain a stable event identifier.

The agent must preserve the event identifier across
delivery attempts.

Cuadrar Cloud will use event identifiers to identify
duplicate submissions within the applicable tenant
and project scope.

The deduplication mechanism must account for concurrent
ingestion requests.

The deduplication retention window and storage strategy
will be defined during Cloud storage architecture design.

Deduplication provides idempotent ingestion behavior
within its documented retention window.

It does not establish exactly-once delivery semantics.

### 9. Authentication

The agent will authenticate with Cuadrar Cloud using
a project-scoped API key.

The API key will be transmitted through the
HTTP Authorization header.

Example:

```http
Authorization: Bearer <api-key>
```

API keys must:

- Be transmitted only over HTTPS.
- Be scoped to a specific project.
- Support revocation.
- Be excluded from application logs.
- Never be included in operational event payloads.

The exact key format, storage mechanism, and rotation
strategy will be defined separately.

### 10. Output Abstraction

The agent will expose an internal output abstraction.

The MVP will support:

#### Cuadrar Cloud Output

Delivers normalized events to Cuadrar Cloud using
the HTTPS ingestion API.

Supports:

- Batching.
- Compression.
- Authentication.
- Retries.
- Delivery acknowledgements.

#### Standard Output

Writes normalized events to standard output.

Primarily intended for:

- Local development.
- Debugging.
- Integration testing.
- Independent agent usage.

The stdout output does not require Cuadrar Cloud
authentication or connectivity.

Additional outputs may be introduced in future releases.

### 11. Compression

The agent will support gzip compression for Cloud delivery.

Compression should be enabled by default for supported
Cloud ingestion requests.

The Cloud API must support compressed and uncompressed
request bodies.

Compression must not bypass request size limits.

The Cloud API must enforce limits on decompressed payload
size to prevent excessive resource consumption.

### 12. Backpressure and Resource Protection

The agent must protect the observed environment from
excessive resource consumption.

The delivery pipeline will enforce bounded resource usage.

Protection mechanisms include:

- Bounded event queues.
- Maximum batch sizes.
- Request timeouts.
- Configurable retry behavior.
- Explicit overflow policies.
- Delivery concurrency limits.

Event collection must not cause uncontrolled memory
growth when Cloud becomes unavailable.

### 13. Agent Observability

The agent must expose basic operational metrics.

At minimum:

| Metric | Description |
|---|---|
| `cuadrar_events_collected_total` | Total collected events |
| `cuadrar_events_dropped_total` | Total dropped events |
| `cuadrar_events_sent_total` | Total successfully delivered events |
| `cuadrar_delivery_failures_total` | Total delivery failures |
| `cuadrar_delivery_retries_total` | Total delivery retry attempts |
| `cuadrar_buffer_events` | Current number of buffered events |

Metrics should be compatible with Prometheus exposition
format.

The agent must also provide structured operational logs.

Logs must not expose authentication credentials or
sensitive event attributes by default.

### 14. Future Extensibility

The transport architecture should allow future support for:

- Persistent disk buffering.
- Additional output destinations.
- Alternative serialization formats.
- Higher-throughput transport protocols.
- Adaptive batching.
- Multiple delivery workers.
- More advanced backpressure strategies.

These capabilities are not required for the initial MVP.

## Alternatives Considered

### 1. Synchronous Event Delivery

Each event is sent immediately after collection.

**Advantages:**

- Simple implementation.
- Minimal delivery latency.
- No batching configuration.

**Disadvantages:**

- High request overhead.
- Poor performance under event bursts.
- Strong coupling between collection and network availability.

**Reason for Rejection:**

Synchronous delivery introduces unnecessary network
overhead and reduces resilience.

### 2. gRPC Streaming

The agent maintains a streaming connection with Cloud.

**Advantages:**

- Efficient binary serialization.
- Lower transport overhead.
- Bidirectional communication capabilities.
- Suitable for high-throughput workloads.

**Disadvantages:**

- Greater implementation complexity.
- More complicated connection lifecycle management.
- Additional infrastructure considerations.
- Unnecessary complexity for initial event volumes.

**Reason for Rejection:**

The MVP does not require the throughput characteristics
or complexity of persistent streaming connections.

### 3. Message Broker-Based Delivery

The agent publishes events directly to a message broker.

**Advantages:**

- Natural asynchronous processing.
- Mature buffering capabilities.
- Potentially strong delivery guarantees.

**Disadvantages:**

- Additional infrastructure dependencies.
- More complicated customer configuration.
- Increased operational requirements.
- Higher adoption barrier.

**Reason for Rejection:**

Requiring a message broker would conflict with the goal
of lightweight agent deployment.

### 4. HTTPS with Asynchronous Batching

The agent sends batches over HTTPS using bounded
buffering and retries.

**Advantages:**

- Simple deployment.
- Broad infrastructure compatibility.
- Efficient enough for initial workloads.
- Straightforward authentication.
- Easy integration with Cloud APIs.

**Disadvantages:**

- Additional buffering logic.
- Duplicate delivery is possible.
- In-memory buffering cannot survive agent restarts.

**Reason for Selection:**

This approach provides an appropriate balance between
simplicity, reliability, and operational efficiency
for the MVP.

## Consequences

### Positive

- The agent does not require persistent connections.
- Cloud delivery remains compatible with standard
  HTTP infrastructure.
- Batching reduces request overhead.
- Retries improve resilience during temporary failures.
- Bounded queues protect agent resource consumption.
- Stable event identifiers support idempotent ingestion.
- Output abstraction enables future integrations.

### Negative

- Events may be lost when the agent restarts.
- In-memory buffering limits outage tolerance.
- At-least-once delivery introduces duplicate submissions.
- Cloud must implement deduplication.
- Batch retries can increase network traffic.
- Buffer overflow requires explicit event dropping.
- HTTP transport may become less efficient at
  significantly higher event volumes.

## MVP Implications

The MVP will implement:

- HTTPS ingestion.
- JSON serialization.
- Asynchronous event batching.
- gzip compression.
- Bounded in-memory buffering.
- Exponential backoff with jitter.
- Project-scoped API key authentication.
- Atomic batch acceptance.
- Stable event identifiers.
- Cloud-side event deduplication.
- Basic agent delivery metrics.
- Cuadrar Cloud and stdout outputs.

The MVP will not implement:

- Persistent disk buffering.
- gRPC streaming.
- Direct Kafka or NATS delivery.
- Exactly-once delivery guarantees.
- Partial batch acceptance.
- Adaptive batching.
- Distributed agent coordination.

## Related Decisions

- ADR-0001: Adopt Unified Operational Events as the Core Data Model
- ADR-0002: Define Open-Source and Cloud Responsibilities
- ADR-0004: Define Agent Architecture (Planned)
- ADR-0005: Define Cloud Ingestion Architecture (Planned)
