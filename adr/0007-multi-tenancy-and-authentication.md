# ADR-0007: Define Multi-Tenancy and Authentication

- **Status:** Proposed
- **Date:** 2026-10-10
- **Decision Makers:** Cuadrar Maintainers

## Context

Cuadrar Cloud is a multi-tenant SaaS platform that allows
independent engineering teams to collect and analyze
operational events.

The platform must securely separate customer data while
providing a simple organization and project management model.

Cuadrar supports two distinct types of clients:

1. Human users accessing the Cloud web application and APIs.
2. Cuadrar Agents submitting operational events.

These clients have different authentication and
authorization requirements.

The architecture must support:

- Secure user authentication.
- Independent organizations.
- Multiple projects per organization.
- Project-scoped event ingestion.
- Tenant data isolation.
- Secure agent credentials.
- Credential revocation.
- Basic organization membership.
- Future role-based access control.
- Independent evolution of authentication providers.

The MVP must avoid unnecessary identity management
complexity while maintaining strong security boundaries.

## Decision

Cuadrar Cloud will use an organization-based multi-tenant
model with project-scoped operational data.

Human users will authenticate through a managed
OpenID Connect-compatible identity provider.

Agents will authenticate using project-scoped API keys.

Authorization and tenant isolation will be enforced
by the Cuadrar Cloud backend.

### 1. Tenant Model

The primary tenant boundary will be the organization.

Each organization may contain multiple projects.

Each project will own its operational events,
agent credentials, and usage records.

Conceptual hierarchy:

```text
Cuadrar Cloud
    |
    +-- Organization A
    |       |
    |       +-- Project 1
    |       |       |
    |       |       +-- Agent Credentials
    |       |       +-- Events
    |       |       +-- Usage Records
    |       |
    |       +-- Project 2
    |               |
    |               +-- Agent Credentials
    |               +-- Events
    |
    +-- Organization B
            |
            +-- Project 3
                    |
                    +-- Agent Credentials
                    +-- Events
```

Organizations will provide the administrative
and future billing boundary.

Projects will provide the primary operational
data isolation boundary.

### 2. Organization Identity

Every organization will have a globally unique
UUID identifier.

An organization will contain:

- Unique identifier.
- Display name.
- Creation timestamp.
- Update timestamp.

Organizations may contain multiple users
and multiple projects.

Organization identifiers must not be derived
from user-provided names.

Organization names may be changed without
affecting organization identity.

### 3. Project Identity

Every project will have a globally unique
UUID identifier.

A project belongs to exactly one organization.

Projects will contain:

- Unique identifier.
- Organization identifier.
- Display name.
- Creation timestamp.
- Update timestamp.

Projects represent isolated operational
environments within an organization.

Examples:

```text
Organization: Acme

Projects:
- Production
- Staging
- Internal Infrastructure
```

Project names are not required to be globally unique.

The MVP may enforce uniqueness within
an individual organization.

### 4. User Identity

Human users will authenticate through
a managed identity provider.

Cuadrar will maintain its own internal
user identifiers.

Each user will have a globally unique UUID.

The internal user identity must remain
independent of the authentication provider.

Conceptual user model:

```text
User
    |
    +-- Internal UUID
    +-- External Identity
    +-- Email
    +-- Display Name
```

External identity mappings will associate
Cuadrar users with identity provider subjects.

The combination of identity provider issuer
and subject must uniquely identify an
external authentication identity.

Email addresses must not be used as
the primary user identifier.

### 5. User Authentication

Cuadrar will use OpenID Connect for
human user authentication.

The authentication provider will handle:

- User login.
- Identity verification.
- Authentication sessions.
- Password management, when applicable.
- Supported social login methods.
- Authentication-related security controls.

Cuadrar Cloud will validate authentication
tokens issued by the configured provider.

Token validation must include:

- Signature verification.
- Issuer validation.
- Audience validation.
- Expiration validation.
- Required claim validation.

The backend must not trust unverified
identity information from the frontend.

The exact authentication provider
will be selected separately.

### 6. Organization Membership

Users may belong to multiple organizations.

Organizations may contain multiple users.

The relationship will be modeled
through organization memberships.

Conceptual structure:

```text
User
  |
  +-- Organization Membership
           |
           +-- Organization
           +-- Role
```

Membership records will contain:

- User identifier.
- Organization identifier.
- Role.
- Creation timestamp.

Organization membership must be verified
before granting access to organization resources.

### 7. Authorization Model

The MVP will implement two organization roles:

- `owner`
- `member`

#### Owner

Organization owners may:

- View organization information.
- Manage organization members.
- Create and manage projects.
- Create and revoke agent credentials.
- View project events.
- Access organization usage information.

#### Member

Organization members may:

- View organization information.
- View projects.
- Explore operational events.
- Access project-level operational information.

Members will not be able to:

- Manage organization membership.
- Delete organizations.
- Create or revoke agent credentials.
- Modify organization-level settings.

The MVP will use organization-level roles.

Project-specific roles and advanced
permission policies are out of scope.

### 8. Authorization Enforcement

Authorization will be enforced by
the Cuadrar Cloud backend.

Frontend visibility rules must not
be considered security boundaries.

Every protected operation must verify:

1. The authenticated user identity.
2. Organization membership.
3. Required organization role.
4. Ownership of the requested resource.

Project access must be restricted
to projects belonging to organizations
accessible by the user.

Authorization checks must not rely
solely on user-provided organization
or project identifiers.

### 9. Agent Authentication

Cuadrar Agents will authenticate
using project-scoped API keys.

Each API key belongs to exactly
one project.

An API key must not grant access
to events belonging to another project.

The MVP will use API keys exclusively
for event ingestion.

Agent credentials must not provide
access to:

- User management.
- Organization administration.
- Project administration.
- Event querying.
- Subscription management.

### 10. API Key Format

API keys will use cryptographically
secure random values.

The format should include a recognizable
prefix to simplify identification
and secret scanning.

Example:

```text
cq_live_<random-secret>
```

The exact format will be defined
during implementation.

API keys must contain sufficient
random entropy to resist guessing.

The key prefix must not be treated
as a security boundary.

### 11. API Key Storage

Raw API keys must not be stored
in the database.

The database will store:

- Credential identifier.
- Project identifier.
- Key hash.
- Display name.
- Creation timestamp.
- Last-used timestamp.
- Revocation timestamp.

API keys will be shown to users
only during creation.

The Cloud backend must not provide
an endpoint for retrieving existing
plaintext API keys.

A cryptographic hash suitable for
high-entropy random tokens will be used.

The exact hashing algorithm will
be selected during implementation.

### 12. API Key Lifecycle

The API key lifecycle will support:

- Creation.
- Authentication.
- Revocation.
- Replacement.

#### Creation

An organization owner may create
a new API key for a project.

The generated secret will be displayed
only once.

#### Authentication

The ingestion API will validate
the provided key and resolve
its associated project.

#### Revocation

An organization owner may revoke
an API key.

Revoked keys must no longer
authorize new ingestion requests.

#### Rotation

Key rotation will use:

1. Create a replacement key.
2. Update the agent configuration.
3. Verify successful ingestion.
4. Revoke the previous key.

Automatic key rotation is not
required for the MVP.

### 13. Agent Identity

API keys identify the project
authorized to receive events.

They do not necessarily identify
a unique agent instance.

Multiple agent instances may use
the same project-scoped credential.

Agent instance identity may be
introduced separately for:

- Agent inventory.
- Health monitoring.
- Deployment tracking.
- Credential auditing.

The MVP must not assume
a one-to-one relationship between
API keys and agent instances.

### 14. Multi-Tenant Data Isolation

Cuadrar will use a shared PostgreSQL
database for the MVP.

Tenant isolation will be enforced
through organization and project
relationships.

Operational events will be associated
with a project identifier.

All event queries must be scoped
to an authorized project.

Example:

```sql
SELECT *
FROM events
WHERE project_id = $1
ORDER BY occurred_at DESC
LIMIT $2;
```

The project identifier must be
validated against the authenticated
user's permissions.

For agent ingestion, the project
identifier must be derived from
the authenticated API key.

### 15. Database Authorization

The application will enforce
tenant isolation through explicit
authorization checks.

Database queries must include
appropriate tenant or project
constraints.

Repository interfaces should
make tenant scoping explicit.

For example:

```go
type EventRepository interface {
    ListByProject(
        ctx context.Context,
        projectID uuid.UUID,
        filter EventFilter,
    ) ([]Event, error)
}
```

Database-level Row-Level Security
may be introduced as an additional
defense mechanism.

RLS is not required for the MVP.

### 16. Authentication Sessions

Human authentication sessions
will be managed through the
selected identity provider
and the Cloud application.

The application must follow
secure session management practices.

Requirements include:

- Secure cookies where applicable.
- HTTPS-only session transport.
- Appropriate SameSite policies.
- Session expiration.
- Protection against CSRF where applicable.
- Protection against token leakage.

The exact frontend authentication
flow will be defined separately.

### 17. Organization Creation

Authenticated users will be able
to create organizations.

The creating user will automatically
receive the `owner` role.

Organization creation must be
performed atomically with
the initial membership assignment.

A user may own multiple organizations.

The MVP may introduce limits on
organization creation to prevent abuse.

### 18. Project Creation

Organization owners may create projects.

Each project must belong to
an existing organization.

Project creation must verify
the user's ownership permissions.

Projects must not be accessible
to users outside their organization.

The MVP may limit the number
of projects per organization.

### 19. Organization Invitations

The architecture will support
future invitation-based membership.

Potential invitation capabilities include:

- Email invitations.
- Invitation expiration.
- Invitation revocation.
- Role assignment.
- Invitation acceptance.

Full invitation management is
not required for the initial MVP.

The MVP may initially support
single-user organizations.

The membership model must not
prevent future multi-user support.

### 20. Auditability

Security-sensitive operations
should produce audit records.

Examples include:

- Organization creation.
- Project creation.
- API key creation.
- API key revocation.
- Membership changes.

Audit records should include:

- Actor identifier.
- Action.
- Target resource.
- Timestamp.
- Relevant non-sensitive metadata.

Raw API keys and authentication
tokens must never be included
in audit records.

A complete enterprise audit
logging system is outside
the MVP scope.

### 21. Security Requirements

The authentication and authorization
architecture must follow these principles:

- Least privilege.
- Secure credential generation.
- No plaintext API key storage.
- Explicit tenant authorization.
- HTTPS-only external communication.
- Secure session handling.
- Credential revocation.
- Protection against cross-tenant access.
- Separation of human and agent authentication.

Security checks must be performed
server-side.

### 22. Testing Strategy

The implementation must include:

- Authentication token validation tests.
- Invalid token rejection tests.
- Expired token rejection tests.
- Organization membership tests.
- Role authorization tests.
- Cross-tenant access tests.
- API key authentication tests.
- API key revocation tests.
- Credential rotation tests.
- Project isolation tests.
- Concurrent organization creation tests.
- Integration tests using PostgreSQL.

Cross-tenant access tests are
mandatory before public release.

## Alternatives Considered

### 1. Single-Tenant Cloud

Each customer receives a dedicated
Cloud deployment and database.

**Advantages:**

- Strong infrastructure isolation.
- Independent customer scaling.
- Simplified tenant-specific configuration.

**Disadvantages:**

- High operational costs.
- Complex deployment management.
- Poor economics for small customers.
- Increased maintenance burden.

**Reason for Rejection:**

Dedicated deployments are not
economically appropriate for the MVP.

### 2. Shared Database with Logical Isolation

All organizations share a database
with explicit tenant-scoped access.

**Advantages:**

- Low infrastructure costs.
- Simple deployment.
- Efficient resource utilization.
- Straightforward initial operations.

**Disadvantages:**

- Requires careful authorization.
- Cross-tenant query bugs are possible.
- Shared database resources may create contention.

**Reason for Selection:**

This approach provides the best
balance between simplicity,
cost, and scalability.

### 3. Custom Authentication System

Cuadrar implements its own
password and session management.

**Advantages:**

- Complete implementation control.
- No external authentication dependency.

**Disadvantages:**

- Significant security responsibility.
- Additional maintenance requirements.
- Increased implementation complexity.
- Slower product development.

**Reason for Rejection:**

Authentication is not a core
differentiating capability of Cuadrar.

### 4. Managed OIDC Authentication

Cuadrar delegates human authentication
to an external identity provider.

**Advantages:**

- Reduced security implementation burden.
- Established authentication flows.
- Support for future identity providers.
- Faster initial development.

**Disadvantages:**

- External service dependency.
- Potential provider costs.
- Integration and configuration requirements.

**Reason for Selection:**

Managed authentication allows
Cuadrar to focus development
resources on its core product.

## Consequences

### Positive

- Clear organization and project boundaries.
- Secure project-scoped agent ingestion.
- Independent human and agent authentication.
- No custom password management.
- Support for multiple organizations per user.
- Simple initial authorization model.
- Low infrastructure overhead.
- Foundation for future team collaboration.
- Explicit cross-tenant security requirements.

### Negative

- Shared database isolation requires discipline.
- Managed authentication introduces an external dependency.
- Organization-level roles limit permission granularity.
- API key rotation requires manual coordination.
- Advanced enterprise identity features are unavailable.
- Authorization checks add complexity to application services.

## MVP Implications

The MVP will implement:

- Organization entities.
- Project entities.
- Internal user identities.
- Managed OIDC authentication.
- Organization ownership.
- Single-user organization support.
- Project-scoped API keys.
- API key creation and revocation.
- Hashed credential storage.
- Backend authorization checks.
- Project-level event isolation.
- Basic security audit records.

The MVP will not implement:

- Enterprise SSO.
- SAML authentication.
- Advanced RBAC.
- Project-specific roles.
- Custom permission policies.
- Automatic API key rotation.
- Dedicated tenant databases.
- Full invitation management.
- Enterprise audit log management.

## Related Decisions

- ADR-0001: Adopt Unified Operational Events as the Core Data Model
- ADR-0002: Define Open-Source and Cloud Responsibilities
- ADR-0003: Define Event Transport and Delivery Semantics
- ADR-0004: Define Agent Architecture
- ADR-0005: Define Cloud Ingestion Architecture
- ADR-0006: Define Event Schema and Identity
