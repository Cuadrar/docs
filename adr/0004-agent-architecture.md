# ADR-0004: Define Agent Architecture

- **Status:** Proposed
- **Date:** 2026-10-10
- **Decision Makers:** Cuadrar Maintainers

## Context

Cuadrar Agent is the open-source component responsible for
collecting, filtering, normalizing, and delivering operational
events from observed environments.

The agent must support multiple event sources while remaining
lightweight, independently deployable, and easy to extend.

The initial implementation will focus on Kubernetes operational
events and a limited set of application-level integrations.

The architecture must satisfy the following requirements:

- Minimal operational complexity.
- Low CPU and memory overhead.
- Explicit event observation.
- Independent operation without Cuadrar Cloud.
- Support for multiple event sources.
- Support for multiple output destinations.
- Bounded resource consumption.
- Reliable delivery within documented limitations.
- Graceful startup and shutdown.
- Straightforward testing and maintenance.

The architecture should allow additional integrations without
requiring fundamental changes to the agent's core processing
pipeline.

## Decision

Cuadrar Agent will be implemented in Go as a modular monolith
using an event-driven processing pipeline.

The agent will consist of independently testable components
connected through well-defined interfaces.

The initial implementation will use a single process with
concurrent event collection and asynchronous delivery.

### 1. High-Level Architecture

The agent will use the following processing pipeline:

```text
                  +----------------------+
                  | Configuration        |
                  +----------------------+
                             |
                             v
+----------------------------------------------------------+
|                      Cuadrar Agent                       |
|                                                          |
|  +----------------------+                                |
|  | Event Collectors     |                                |
|  |                      |                                |
|  | - Kubernetes         |                                |
|  | - Application        |                                |
|  | - Future Sources     |                                |
|  +----------------------+                                |
|             |                                            |
|             v                                            |
|  +----------------------+                                |
|  | Event Filtering      |                                |
|  +----------------------+                                |
|             |                                            |
|             v                                            |
|  +----------------------+                                |
|  | Event Normalization  |                                |
|  +----------------------+                                |
|             |                                            |
|             v                                            |
|  +----------------------+                                |
|  | Event Enrichment     |                                |
|  +----------------------+                                |
|             |                                            |
|             v                                            |
|  +----------------------+                                |
|  | Bounded Event Queue  |                                |
|  +----------------------+                                |
|             |                                            |
|             v                                            |
|  +----------------------+                                |
|  | Output Dispatcher    |                                |
|  +----------------------+                                |
|             |                                            |
|       +-----+-----+                                      |
|       |           |                                      |
|       v           v                                      |
|  +---------+ +---------+                                 |
|  | Cuadrar | | stdout  |                                 |
|  | Cloud   | |         |                                 |
|  +---------+ +---------+                                 |
|                                                          |
+----------------------------------------------------------+
```

Collectors, processors, and outputs will be implemented
as internal Go components.

The MVP does not require dynamically loaded plugins.

### 2. Agent Components

#### 2.1 Collectors

Collectors are responsible for observing supported systems
and producing source-specific events.

Each collector will:

- Initialize its source integration.
- Establish the required watches or subscriptions.
- Observe configured event types.
- Extract relevant source metadata.
- Forward collected events into the processing pipeline.
- Handle source-specific reconnection and recovery.
- Stop gracefully when requested.

Collectors must not communicate directly with Cuadrar Cloud.

Collectors must not implement cloud-specific authentication,
batching, or delivery logic.

The agent will support multiple collectors running concurrently.

#### 2.2 Event Filtering

The filtering component determines whether a collected event
should continue through the processing pipeline.

Filtering will follow an explicit observation model.

Only configured event categories and types will be accepted.

Filtering should occur as early as practical to avoid
unnecessary processing.

The filtering system may support:

- Event type selection.
- Kubernetes namespace selection.
- Resource type selection.
- Label-based filtering.
- Source-specific filtering rules.

The exact filtering configuration will be defined separately.

#### 2.3 Event Normalization

Normalization converts source-specific events into the
Unified Operational Event model defined by ADR-0001.

The normalization component will:

- Map source-specific event types to canonical event types.
- Populate required event attributes.
- Preserve supported source metadata.
- Validate required event fields.
- Generate or assign stable event identifiers.

Normalization must not depend on Cuadrar Cloud.

The canonical event schema and identifier generation
strategy will be defined separately.

#### 2.4 Event Enrichment

Enrichment adds contextual information to normalized events.

Examples include:

- Service identification.
- Environment identification.
- Kubernetes namespace.
- Cluster identifier.
- Resource metadata.

Enrichment must not introduce uncontrolled network calls
or significant processing overhead.

Source-specific enrichment should use locally available
information whenever possible.

Cloud-side enrichment remains outside the agent's scope.

#### 2.5 Event Queue

The agent will use a bounded in-memory queue to decouple
event collection from event delivery.

The queue must:

- Enforce a configurable capacity.
- Support concurrent event producers.
- Support controlled consumption.
- Expose queue utilization metrics.
- Apply the configured overflow policy.

The default overflow policy will be to drop newly arriving
events when the queue is full.

Persistent disk buffering is not included in the MVP.

#### 2.6 Output Dispatcher

The output dispatcher will consume normalized events
from the queue and forward them to configured outputs.

Responsibilities include:

- Managing output workers.
- Dispatching events to destinations.
- Applying destination-specific batching.
- Coordinating delivery retries.
- Handling delivery failures.
- Reporting delivery metrics.

The dispatcher must not contain source-specific
collection logic.

The MVP will support:

- Cuadrar Cloud output.
- Standard output.

The initial implementation may support one active
destination at a time.

Support for simultaneous delivery to multiple outputs
may be introduced later.

### 3. Component Interfaces

Components will communicate through Go interfaces.

Interfaces should remain small and focused on
their responsibilities.

The following definitions are conceptual examples.

#### Collector

```go
type Collector interface {
    Run(ctx context.Context, emit func(RawEvent) error) error
}
```

A collector runs until its context is cancelled or
an unrecoverable error occurs.

The emit function forwards source-specific events
into the processing pipeline.

#### Processor

```go
type Processor interface {
    Process(ctx context.Context, event Event) (Event, bool, error)
}
```

The boolean return value indicates whether the event
should continue through the pipeline.

Processors may implement filtering, normalization,
or enrichment.

The exact interface may evolve during implementation.

#### Output

```go
type Output interface {
    WriteBatch(ctx context.Context, events []Event) error
    Close(ctx context.Context) error
}
```

Outputs are responsible for destination-specific delivery.

Batching and retry coordination may be implemented by
the dispatcher or shared delivery components.

The final interface boundaries will be determined
during implementation.

### 4. Concurrency Model

The agent will use Go goroutines and bounded channels
for concurrent processing.

The concurrency model will follow these principles:

- Collectors may run concurrently.
- Event processing must remain bounded.
- Output delivery must not block source observation indefinitely.
- Shared state must be protected against data races.
- All background goroutines must support cancellation.
- The agent must avoid unbounded goroutine creation.

The initial implementation will use a fixed number
of processing and delivery workers.

Worker counts may become configurable based on
performance requirements.

### 5. Kubernetes Integration

The initial Kubernetes collector will use the
official Kubernetes Go client.

The collector will interact with the Kubernetes API
using supported watch mechanisms.

Where appropriate, shared informers may be used
to reduce API server load.

The collector should support:

- Kubernetes Event resources.
- Pod lifecycle observations.
- Deployment-related changes.
- Selected workload state transitions.

The exact supported event types will be defined
in the MVP Product Scope.

The collector must not require privileged access
to Kubernetes nodes.

### 6. Application Event Receiver

The agent will support application-level event ingestion through
an HTTP/JSON receiver.

The receiver will allow applications to emit operational events
without requiring language-specific SDKs.

#### Transport

The MVP will use HTTP with JSON serialization.

The initial endpoint will be:

```http
POST /v1/events
Content-Type: application/json
```

The receiver will accept normalized operational events or
application-specific events that can be mapped to the unified
event model.

The exact request schema will be defined separately.

#### Responsibilities

The Application Event Receiver will:

- Accept incoming HTTP requests.
- Validate request payloads.
- Enforce request size limits.
- Convert incoming payloads into internal event representations.
- Apply configured observation rules.
- Forward accepted events into the shared processing pipeline.
- Return appropriate HTTP responses.

The receiver must not implement Cloud-specific delivery logic.

#### Security

The receiver will bind to localhost by default.

External access must require explicit configuration.

The receiver must support configurable request size limits
and timeouts.

Authentication and network exposure requirements for
non-local deployments will be defined before enabling
remote ingestion.

#### Future gRPC Support

The receiver architecture must remain transport-independent.

HTTP handlers must not contain business logic related to
event filtering, normalization, enrichment, or delivery.

Future versions may introduce a gRPC receiver using
Protocol Buffers.

Both HTTP and gRPC receivers will use the same internal
event processing pipeline.

Conceptual architecture:

```text
Application
    |
    +--------------------+
    |                    |
    v                    v
HTTP/JSON            gRPC/Protobuf
(MVP)                (Future)
    |                    |
    v                    v
+----------------------------+
| Application Event Ingestion|
+----------------------------+
              |
              v
+----------------------------+
| Unified Event Pipeline     |
|                            |
| Filtering                  |
| Normalization              |
| Enrichment                 |
| Buffering                  |
+----------------------------+
              |
              v
+----------------------------+
| Output Dispatcher          |
+----------------------------+
```

The addition of gRPC must not require changes to existing
collectors, processors, or outputs.

### 7. Kubernetes Deployment

The MVP agent will run as a Kubernetes Deployment.

The initial deployment model will use a single replica.

This avoids duplicate event collection caused by
multiple agents observing the same cluster.

The agent will use Kubernetes RBAC with the minimum
permissions required by enabled collectors.

The deployment must support:

- ConfigMap-based configuration.
- Secret-based credentials.
- Resource requests and limits.
- Liveness and readiness probes.
- Graceful termination.
- Service account configuration.

A Helm chart may be provided to simplify installation.

DaemonSet deployment is not required for the MVP.

### 8. Configuration

The agent will use YAML as its primary configuration format.

Environment variables may be used for configuration
overrides and sensitive values.

Example:

```yaml
agent:
  name: cuadrar-agent

collectors:
  kubernetes:
    enabled: true
    namespaces:
      - default
      - production

    events:
      - pod.restarted
      - deployment.updated
      - pod.failed

pipeline:
  buffer:
    capacity: 10000
    overflow_policy: drop_newest

outputs:
  cuadrar:
    enabled: true
    endpoint: https://api.cuadrar.dev
    api_key: ${CUADRAR_API_KEY}

    batch:
      max_events: 100
      flush_interval: 5s

  stdout:
    enabled: false
```

This configuration is illustrative.

The final configuration schema will be defined
during implementation.

### 9. Lifecycle Management

The agent will have an explicit lifecycle.

#### Startup

During startup, the agent will:

1. Load configuration.
2. Validate configuration.
3. Initialize metrics and logging.
4. Initialize collectors.
5. Initialize processing components.
6. Initialize configured outputs.
7. Start background workers.
8. Begin event collection.

Startup must fail when required configuration
is invalid or critical components cannot initialize.

#### Shutdown

During graceful shutdown, the agent will:

1. Stop accepting new events.
2. Cancel active collectors.
3. Allow in-flight processing to finish.
4. Attempt to flush buffered events.
5. Close configured outputs.
6. Stop health and metrics endpoints.

Shutdown must respect a configurable timeout.

Events remaining after the shutdown deadline
may be discarded.

The agent does not guarantee durable event
preservation across restarts.

### 10. Health Checks

The agent will expose HTTP health endpoints.

Proposed endpoints:

```http
GET /healthz
GET /readyz
```

`/healthz` indicates whether the agent process
is functioning.

`/readyz` indicates whether the agent has
successfully initialized its required components
and is ready to operate.

Temporary Cuadrar Cloud unavailability should not
automatically cause the agent to restart.

Delivery failures should be exposed through metrics
rather than triggering unnecessary Kubernetes restarts.

### 11. Metrics and Logging

The agent will expose Prometheus-compatible metrics.

The metrics endpoint will be:

```http
GET /metrics
```

The agent will use structured logging.

Logs should include:

- Component name.
- Event source.
- Operational status.
- Error details.
- Relevant non-sensitive metadata.

Logs must not expose:

- API keys.
- Authentication tokens.
- Application secrets.
- Sensitive event payloads by default.

### 12. Error Handling

Errors will be classified according to their scope.

#### Collector Errors

Temporary collector failures should trigger
source-specific recovery or reconnection.

Unrecoverable collector failures may cause
the agent to terminate when the affected collector
is required for operation.

#### Processing Errors

Invalid or unsupported events should be dropped
with appropriate metrics.

A malformed event must not terminate the
entire agent process.

#### Delivery Errors

Delivery errors will be handled according
to ADR-0003.

Temporary destination failures must not
cause uncontrolled resource consumption.

### 12. Testing Strategy

The agent architecture must support testing
without requiring a live Cuadrar Cloud instance.

Testing will include:

- Unit tests for collectors and processors.
- Unit tests for filtering and normalization.
- Output tests using mock HTTP servers.
- Configuration validation tests.
- Queue overflow tests.
- Retry and timeout tests.
- Graceful shutdown tests.
- Kubernetes integration tests.
- Race detection tests.

The project should use standard Go testing
tools wherever possible.

### 13. Extensibility

New collectors should be implemented without
modifying the core delivery pipeline.

New outputs should be implemented without
modifying event collection logic.

The architecture should support future additions
such as:

- Kafka collectors.
- NATS collectors.
- Database integrations.
- HTTP application event receivers.
- Custom event sources.
- Persistent buffering.
- Additional output destinations.
- Multiple agent replicas with coordinated collection.

These capabilities are not required for the MVP.

## Alternatives Considered

### 1. Monolithic Agent Without Component Boundaries

All collection, processing, and delivery logic
is implemented in a tightly coupled application.

**Advantages:**

- Minimal initial structure.
- Faster implementation for a single event source.

**Disadvantages:**

- Difficult to extend.
- Harder to test independently.
- Increased coupling between integrations.
- Greater maintenance complexity.

**Reason for Rejection:**

Cuadrar is expected to support multiple event sources
and destinations.

A tightly coupled implementation would create
unnecessary long-term maintenance costs.

### 2. Microservice-Based Agent

Event collection, processing, and delivery are
implemented as separate services.

**Advantages:**

- Independent component scaling.
- Strong process isolation.
- Flexible deployment options.

**Disadvantages:**

- Significantly higher operational complexity.
- Additional network communication.
- Increased resource consumption.
- More complicated deployment and debugging.

**Reason for Rejection:**

A distributed agent architecture would conflict
with the requirement for lightweight deployment.

### 3. Dynamic Plugin Architecture

Collectors and outputs are loaded dynamically
at runtime.

**Advantages:**

- Independent plugin distribution.
- Runtime extensibility.
- Potential third-party integration ecosystem.

**Disadvantages:**

- Additional compatibility requirements.
- Increased security considerations.
- More complex lifecycle management.
- Higher implementation and testing costs.

**Reason for Rejection:**

Compile-time component registration provides
sufficient extensibility for the MVP.

### 4. Modular Monolith with Event Pipeline

A single Go process contains independently
testable collectors, processors, and outputs.

**Advantages:**

- Simple deployment.
- Clear component boundaries.
- Low operational overhead.
- Straightforward testing.
- Efficient in-process communication.
- Incremental extensibility.

**Disadvantages:**

- Components share process resources.
- Individual components cannot scale independently.
- Fault isolation is limited to application-level handling.

**Reason for Selection:**

This approach provides the best balance between
simplicity, performance, and extensibility.

## Consequences

### Positive

- The agent can run as a single lightweight process.
- Collectors and outputs can evolve independently.
- The architecture supports multiple event sources.
- Bounded queues protect resource consumption.
- Go provides efficient concurrency primitives.
- Kubernetes integration can use established client libraries.
- The agent remains independent of Cuadrar Cloud.
- Components can be tested in isolation.

### Negative

- A single process provides limited fault isolation.
- In-memory queues do not survive agent restarts.
- Concurrent processing requires careful synchronization.
- A single Kubernetes replica limits collection scalability.
- Shared process resources may create contention.
- Additional integrations increase the agent binary size.

## MVP Implications

The MVP will implement:

- Go-based agent.
- Modular monolith architecture.
- Collector interfaces.
- Processing pipeline.
- Explicit event filtering.
- Unified event normalization.
- Basic event enrichment.
- HTTP/JSON Application Event Receiver.
- Configurable HTTP receiver address and port.
- Application event payload validation.
- Request size limits and timeouts.
- Shared processing pipeline for Kubernetes and application events.
- Bounded in-memory queue.
- Output abstraction.
- Cuadrar Cloud output.
- Standard output.
- Kubernetes collector.
- YAML configuration.
- Structured logging.
- Prometheus-compatible metrics.
- Health and readiness endpoints.
- Graceful shutdown.
- Kubernetes Deployment configuration.

The MVP will not implement:

- Dynamic plugin loading.
- gRPC Application Event Receiver.
- Protocol Buffers-based application ingestion.
- Language-specific application SDKs.
- OpenTelemetry Protocol (OTLP) receiver.
- Multiple coordinated agent replicas.
- Persistent disk buffering.
- DaemonSet-based collection.
- Distributed processing.
- Runtime component installation.
- Independent component scaling.

## Related Decisions

- ADR-0001: Adopt Unified Operational Events as the Core Data Model
- ADR-0002: Define Open-Source and Cloud Responsibilities
- ADR-0003: Define Event Transport and Delivery Semantics
- ADR-0005: Define Cloud Ingestion Architecture (Planned)
