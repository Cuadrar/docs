# ADR-0002: Define Open-Source and Cloud Responsibilities

- **Status:** Accepted
- **Date:** 2026-10-10
- **Decision Makers:** Cuadrar Maintainers

## Context

Cuadrar follows an open-source-first product strategy.

The platform consists of two primary architectural domains:

1. An open-source agent responsible for observing operational events within customer environments.
2. A managed cloud platform responsible for storing, exploring, and analyzing collected events.

A clear separation of responsibilities is required to ensure that:

- The open-source agent remains independently useful.
- Customers retain control over event collection and transmission.
- The cloud platform can evolve independently of agent releases.
- Commercial capabilities do not introduce unnecessary limitations into the open-source implementation.
- The system remains extensible and avoids unnecessary vendor lock-in.

This separation must also establish a sustainable foundation for Cuadrar's SaaS business model.

## Decision

Cuadrar will adopt a **hybrid open-source and proprietary cloud architecture**.

The event collection layer will be open source, while the managed Cuadrar Cloud platform will remain proprietary.

The architectural boundary will be defined by responsibility rather than arbitrary feature restrictions.

### 1. Open-Source Agent

The Cuadrar Agent will be developed and distributed as open-source software.

Its primary responsibility is to observe, normalize, filter, and deliver operational events.

The agent will operate within infrastructure controlled by the user.

#### Responsibilities

The agent will be responsible for:

- Collecting events from supported sources.
- Applying explicit observation configuration.
- Filtering events before transmission.
- Normalizing collected events into the unified event model.
- Enriching events with available source metadata.
- Batching events when appropriate.
- Delivering events to configured destinations.
- Handling temporary delivery failures.
- Exposing basic operational health information.

#### Independence

The agent must not require Cuadrar Cloud to perform its core responsibilities.

Users must be able to configure alternative event destinations through supported output interfaces.

The agent must not require a Cuadrar account, license key, or cloud connection to operate with non-Cloud destinations.

#### Configuration

The agent will use declarative configuration to define:

- Enabled event sources.
- Observed event types.
- Filtering rules.
- Destination configuration.
- Authentication settings.
- Resource usage limits, where supported.

The initial configuration format will be determined during agent architecture design.

#### Extensibility

The agent architecture should support additional event sources and destinations without fundamental changes to its core processing pipeline.

The MVP does not require a runtime plugin system or dynamically loaded extensions.

### 2. Cuadrar Cloud

Cuadrar Cloud will be a proprietary, managed SaaS platform.

Its primary responsibility is to transform collected operational events into accessible and actionable operational information.

#### Responsibilities

Cuadrar Cloud will be responsible for:

- Authenticating incoming agent requests.
- Validating and ingesting operational events.
- Associating events with organizations and projects.
- Storing events according to configured retention policies.
- Providing event search and filtering.
- Exposing APIs for event retrieval.
- Providing a web interface for event exploration.
- Visualizing relationships between services, resources, and events.
- Managing organizations, projects, and agent credentials.
- Enforcing subscription limits and usage policies.

#### Future Capabilities

The cloud architecture should allow future introduction of:

- Advanced event correlation.
- Incident timelines.
- Automated anomaly detection.
- Alerting and notifications.
- AI-assisted incident investigation.
- Historical operational analysis.
- Team collaboration.
- Enterprise authentication and access control.

These capabilities are not automatically included in the MVP.

### 3. Responsibility Matrix

| Capability | OSS Agent | Cuadrar Cloud |
|---|---|---|
| Event collection | Yes | No |
| Source integrations | Yes | No |
| Event filtering | Yes | No |
| Event normalization | Yes | Validation only |
| Event enrichment | Source-level | Cloud-level |
| Event batching | Yes | No |
| Event delivery | Yes | Receives events |
| Delivery retries | Yes | No |
| Event storage | Temporary buffering only | Yes |
| Event search | No | Yes |
| Event visualization | No | Yes |
| Service relationship visualization | No | Yes |
| Organization management | No | Yes |
| Project management | No | Yes |
| Agent credential management | No | Yes |
| Subscription management | No | Yes |
| Advanced event correlation | No | Future |
| AI-powered investigation | No | Future |

### 4. Event Processing Boundary

The agent will process events locally before transmitting them to a configured destination.

The conceptual data flow will be:

```text
Observed Environment
        |
        v
+----------------------+
| Event Sources        |
|                      |
| Kubernetes           |
| Applications         |
| Messaging Systems    |
| Custom Integrations  |
+----------------------+
        |
        v
+----------------------+
| Cuadrar Agent        |
|                      |
| Collection           |
| Filtering            |
| Normalization        |
| Enrichment           |
| Batching             |
+----------------------+
        |
        v
+----------------------+
| Output Interface     |
+----------------------+
        |
        +--------------------------+
        |                          |
        v                          v
+-------------------+    +-------------------+
| Cuadrar Cloud     |    | Custom Destination|
|                   |    |                   |
| Ingestion         |    | External Systems  |
| Storage           |    |                   |
| Analysis          |    |                   |
| Visualization     |    |                   |
+-------------------+    +-------------------+
```

The output interface will provide a stable abstraction between event processing and event delivery.

The initial implementation will support two destinations:

- `cuadrar`: Sends normalized events to Cuadrar Cloud.
- `stdout`: Writes normalized events to standard output for local development and testing.

Additional destinations, such as Kafka, NATS, or OpenTelemetry-compatible systems, may be introduced in future releases.

### 5. Data Ownership and Privacy

Operational events may contain sensitive infrastructure and application metadata.

Cuadrar will follow a data minimization approach.

The agent must allow users to control which events are collected and transmitted.

Event collection must not implicitly include:

- Application secrets.
- Authentication tokens.
- Environment variable values.
- Request or response bodies.
- Unrestricted application logs.

Source integrations must explicitly define which information they collect.

Users remain responsible for configuring observation rules appropriate to their environments.

Cuadrar Cloud will process received event data according to its published data handling and retention policies.

### 6. Compatibility and Versioning

The agent and cloud platform will be released independently.

The event ingestion interface must support compatibility across supported agent versions.

The system should not require simultaneous upgrades of the agent and cloud platform.

Breaking changes to event schemas or ingestion protocols must follow an explicit versioning and migration strategy.

The exact compatibility guarantees will be defined separately.

### 7. Licensing

The Cuadrar Agent will be distributed under an open-source license.

The specific license will be selected before the first public release.

Cuadrar Cloud will remain proprietary.

The licensing model must not prevent users from:

- Running the agent independently.
- Inspecting and modifying agent source code.
- Building integrations with supported agent interfaces.
- Sending collected events to alternative destinations.

The selected open-source license will define the precise redistribution and modification rights.

## Alternatives Considered

### 1. Fully Proprietary Platform

Both the agent and cloud platform remain closed source.

**Advantages:**

- Complete control over the codebase.
- Simpler commercial licensing.
- Reduced exposure of implementation details.

**Disadvantages:**

- Lower transparency for infrastructure software.
- Reduced community contribution opportunities.
- Greater concerns about vendor lock-in.
- Higher adoption barriers.

**Reason for Rejection:** This approach conflicts with Cuadrar's open-source-first strategy.

### 2. Fully Open-Source Platform

Both the agent and cloud platform are open source.

Revenue would primarily come from managed hosting and enterprise services.

**Advantages:**

- Maximum transparency.
- Self-hosting capabilities.
- Strong community participation opportunities.

**Disadvantages:**

- Additional maintenance burden.
- More complex commercial differentiation.
- Increased support requirements for self-hosted deployments.
- Greater initial development scope.

**Reason for Rejection:** Supporting a complete self-hosted platform would significantly increase the initial scope and operational complexity.

### 3. Open-Source Agent with Proprietary Cloud

The collection layer is open source, while managed analysis and visualization capabilities remain proprietary.

**Advantages:**

- Low adoption barrier.
- Clear separation of responsibilities.
- Independent development and release cycles.
- Reduced scope for the initial product.
- Sustainable SaaS monetization opportunities.

**Disadvantages:**

- Cloud functionality cannot be fully self-hosted.
- Users depend on Cuadrar Cloud for native visualization and analysis.
- Public interfaces require careful compatibility management.

**Reason for Selection:** This approach provides the best balance between open-source adoption, product focus, and commercial sustainability.

## Consequences

### Positive

- The agent can be adopted without requiring a Cuadrar Cloud subscription.
- The cloud platform can evolve independently of event collection.
- Users maintain control over event observation and delivery.
- New event sources can benefit both cloud users and independent agent users.
- Commercial differentiation is based on managed capabilities rather than artificially restricted collection features.
- The architecture supports gradual expansion of the integration ecosystem.

### Negative

- Public event schemas and output interfaces require long-term compatibility management.
- Maintaining an open-source agent introduces community support and documentation responsibilities.
- Independent agent deployments may expose integration issues that are unrelated to Cuadrar Cloud.
- Cloud-specific functionality must remain outside the open-source agent's core processing logic.
- Supporting alternative destinations increases the agent's testing and maintenance scope.

## MVP Implications

### Open-Source Agent

The initial MVP will include:

- Kubernetes event collection.
- A limited set of application-level event integrations.
- Explicit event filtering.
- Unified event normalization.
- Cuadrar Cloud output.
- Standard output (`stdout`) destination.
- Basic delivery retries and operational health reporting.

### Cuadrar Cloud

The initial MVP will include:

- Event ingestion API.
- Organization and project isolation.
- Agent authentication.
- Event storage.
- Basic event search and filtering.
- Event exploration interface.
- Initial service and resource relationship visualization.

### Out of Scope

The MVP will not include:

- A self-hosted version of Cuadrar Cloud.
- Advanced event correlation.
- AI-powered incident investigation.
- Enterprise SSO.
- Advanced role-based access control.
- A general-purpose runtime plugin marketplace.
- Additional event destinations beyond `cuadrar` and `stdout`.

## Related Decisions

- ADR-0001: Adopt Unified Operational Events as the Core Data Model
- ADR-0003: Define Event Transport and Delivery Semantics (Planned)
- ADR-0004: Define Agent Architecture (Planned)
