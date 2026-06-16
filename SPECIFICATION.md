# Nscale Cloud Platform — Unified Architectural Specification

**Version 3.23 | March 2026**
Nscale Cloud Platform Engineering

*Audience: Engineers · AI Automation Agents · Platform Contributors*
*Services: Identity · Region · Compute · Kubernetes · Core*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Platform Model](#2-platform-model)
   - [2.1 Services and Owned Resources](#21-services-and-owned-resources)
   - [2.2 Service Dependency Order](#22-service-dependency-order)
   - [2.3 Cloud Provider Abstraction](#23-cloud-provider-abstraction)
   - [2.4 Service Layering and Resource Accountability](#24-service-layering-and-resource-accountability)
3. [Data Boundaries](#3-data-boundaries)
   - [3.1 Authority and Derived State](#31-authority-and-derived-state)
4. [Identity and Access](#4-identity-and-access)
   - [4.1 Organisational Hierarchy](#41-organisational-hierarchy)
   - [4.2 Actors and Representation](#42-actors-and-representation)
   - [4.3 RBAC](#43-rbac)
   - [4.4 Principals and Proxies](#44-principals-and-proxies)
   - [4.5 Principal Propagation](#45-principal-propagation)
     - [4.5.1 API Authentication Classifications](#451-api-authentication-classifications)
   - [4.6 Principal Propagation Modes and Impersonation](#46-principal-propagation-modes-and-impersonation)
     - [4.6.1 ACL Intersection under Impersonation](#461-acl-intersection-under-impersonation)
   - [4.7 Federated Authentication](#47-federated-authentication)
   - [4.8 Token Infrastructure](#48-token-infrastructure)
5. [Resource Graph](#5-resource-graph)
   - [5.1 Edge Properties](#51-edge-properties)
   - [5.2 Edge Types by Property Intersection](#52-edge-types-by-property-intersection)
   - [5.3 References](#53-references)
   - [5.4 Event Bus](#54-event-bus)
   - [5.5 Containment](#55-containment)
   - [5.6 Consumption](#56-consumption)
   - [5.7 Dependency](#57-dependency)
   - [5.8 Status Propagation](#58-status-propagation)
   - [5.9 Deletion Propagation Mechanisms](#59-deletion-propagation-mechanisms)
   - [5.10 Defining a New Resource Type](#510-defining-a-new-resource-type)
   - [5.11 Deletion Semantics](#511-deletion-semantics)
   - [5.12 Deletion State Machine](#512-deletion-state-machine)
   - [5.13 Deletion Events](#513-deletion-events)
6. [Resource Model](#6-resource-model)
   - [6.1 Naming and Metadata](#61-naming-and-metadata)
   - [6.2 Labels and Annotations](#62-labels-and-annotations)
   - [6.3 Tags](#63-tags)
7. [API Design](#7-api-design)
   - [7.1 Synchronous vs Asynchronous Operations](#71-synchronous-vs-asynchronous-operations)
   - [7.2 Handler Layer Responsibilities](#72-handler-layer-responsibilities)
     - [7.2.1 Handler Authentication Classification](#721-handler-authentication-classification)
     - [7.2.2 Server Middleware Stack](#722-server-middleware-stack)
   - [7.3 Conflict Detection](#73-conflict-detection)
   - [7.4 Resource References](#74-resource-references)
   - [7.5 Multi-Step Operations (Sagas)](#75-multi-step-operations-sagas)
   - [7.6 Quota and Strongly Consistent Allocations](#76-quota-and-strongly-consistent-allocations)
   - [7.7 API Versioning](#77-api-versioning)
   - [7.8 List Filtering](#78-list-filtering)
   - [7.9 Error Handling and Propagation](#79-error-handling-and-propagation)
   - [7.10 v2 API Design Model](#710-v2-api-design-model)
   - [7.11 OpenAPI-First Development](#711-openapi-first-development)
8. [Controller Behaviour](#8-controller-behaviour)
   - [8.1 The Work Queue](#81-the-work-queue)
   - [8.2 Transient Conditions and Silent Retry](#82-transient-conditions-and-silent-retry)
   - [8.3 Genuine Errors](#83-genuine-errors)
   - [8.4 Status Conditions](#84-status-conditions)
   - [8.5 Finalizer Lifecycle](#85-finalizer-lifecycle)
   - [8.6 Controller Watches](#86-controller-watches)
   - [8.7 Downstream Error Handling](#87-downstream-error-handling)
   - [8.8 Deletion Deadlock Detection](#88-deletion-deadlock-detection)
   - [8.9 Status Projection and Monitoring](#89-status-projection-and-monitoring)
   - [8.10 Authoritative State for Deprovisioning](#810-authoritative-state-for-deprovisioning)
9. [Observability](#9-observability)
   - [9.1 Structured Logging](#91-structured-logging)
   - [9.2 Audit Logging](#92-audit-logging)
   - [9.3 Distributed Tracing](#93-distributed-tracing)
   - [9.4 Metrics](#94-metrics)
10. [Security](#10-security)
    - [10.1 Platform Security Invariants](#101-platform-security-invariants)
11. [Core Library](#11-core-library)
12. [Glossary](#12-glossary)

**Appendix**

- [A.1 TOCTOU Race — Resource Name Uniqueness](#a1-toctou-race--resource-name-uniqueness)
- [A.2 Pagination — Known Deficiency](#a2-pagination--known-deficiency)
- [A.3 Quota Allocation — Async Boundary Violation](#a3-quota-allocation--async-boundary-violation)
- [A.4 Update Saga Revert — Broken Compensation](#a4-update-saga-revert--broken-compensation)
- [A.5 Multi-Step Create With No Saga — Orphaned Cross-Service Resources](#a5-multi-step-create-with-no-saga--orphaned-cross-service-resources)
- [A.6 Allocation Ownership Inversion](#a6-allocation-ownership-inversion)
- [A.7 Best-Effort Status Used as Deprovision Authority](#a7-best-effort-status-used-as-deprovision-authority)

**Appendix B**

- [B.1 Checklist: Building a New Service](#b1-checklist-building-a-new-service)

---

## 1. Purpose

This document defines the rules of the Nscale Cloud Platform. It is the normative reference for engineers and AI automation agents. Any code or design that does not conform to these rules is a defect and must be fixed or deleted.

> **Rationale:** The platform is built and extended by multiple engineers and automated agents over time. Without a shared, normative reference, each contributor makes local decisions that are locally consistent but globally contradictory. The result is a codebase where the same problem is solved five different ways, none of them quite right. This document exists to prevent that. It is the single source of truth for how the platform works.

---

## 2. Platform Model

The Nscale Cloud Platform is a cloud-agnostic PaaS for AI/ML workloads. Each service exposes a versioned REST API over HTTPS and owns a persistent store for the resources it manages. Service-to-service communication is exclusively mTLS — the calling service's certificate is its identity. The shared library (referred to throughout as the core library) is not a deployed service.

> **Rationale:** The platform is composed of independent services, each responsible for a specific domain. Knowing which service owns which resources — and in what order they depend on each other — is essential before making any cross-service design decision. A service that tries to own resources outside its domain, or that calls a downstream dependency before the upstream is ready, will produce inconsistent state that is difficult to recover from.

### 2.1 Services and Owned Resources

| Service | Role | Owns |
|---|---|---|
| Identity | Identity Service | Organisation, Project, User, OrganizationUser, Group, Role, ServiceAccount, OAuth2Provider, OAuth2Client, Quota, Allocation, SigningKey |
| Region | Region Service | Region, Identity, Network, SecurityGroup, LoadBalancer, SSHCertificateAuthority, Server, FileStorage |
| Compute | Compute Service | ComputeInstance |
| Kubernetes | Kubernetes Service | ClusterManager, Cluster |
| Core | Core Library | (shared library — no resources) |

### 2.2 Service Dependency Order

```
Identity Service  ←  Region Service  ←  Compute Service
                                     ←  Kubernetes Service
```

### 2.3 Cloud Provider Abstraction

The Region service implements the platform's cloud-agnosticism through a provider interface. All cloud lifecycle operations — identity provisioning, network management, server lifecycle, image and flavor discovery — are expressed against this interface. No platform code above the Region service has any knowledge of the underlying cloud substrate.

Two capability levels exist. The discovery interface covers region capability and flavor inventory and must be implemented by all providers. The full lifecycle interface extends this with the complete set of cloud operations and is required for providers that host user workloads.

Three provider implementations exist: OpenStack (full lifecycle), Kubernetes (discovery only, for Kubernetes-backed regions), and a simulated backend used for testing. New cloud substrates are onboarded by implementing the provider interface; no changes to Region's handler or controller logic are required.

Where a cloud API is insufficient — for example, where the provider does not allocate network segmentation IDs or where image metadata retrieval is too slow for interactive use — the Region service implements compensating mechanisms at the provider layer. These are internal to Region and invisible to callers.

### 2.4 Service Layering and Resource Accountability

Higher-order services build on lower-order services by following a consistent layering pattern: the higher-order service owns its own resource (enforcing the data boundary), drives the lower-order service's resource through its versioned API, and projects status back up from the lower-order resource into its own.

The lower-order resource is an implementation detail of the higher-order service. It is not exposed to users and is not user-managed. Users interact only with the resource the higher-order service owns.

**Quota and billing accountability is conferred by the service layer, not by the underlying primitive.** The same lower-order resource can be either user-accountable or a platform-managed resource carrying no quota or billing significance, depending on which service provisioned it:

- The Compute service provisions `region.Server` resources via the Region API and wraps them in `ComputeInstance`. Quota is charged and billing is attributed at the `ComputeInstance` layer. The `region.Server` itself carries no billing significance.
- The Kubernetes service provisions `region.Server` resources directly via the Region API for cluster nodes. These servers are scoped to the organisation but bypass the Compute accounting layer entirely — no quota is charged and no billing applies. They are platform-managed infrastructure.

The rule for new services follows directly: if the infrastructure being provisioned is platform-managed and not directly requested by the user, provision it directly from Region. If it is user-requested and user-accountable, wrap it in a service-owned resource that carries the quota and billing responsibility.

> **Rationale:** Placing accountability at the primitive layer would mean all servers — whether user-requested or platform-managed — would consume quota. That would make Kubernetes cluster nodes count against user quota, which is wrong. Placing accountability at the service layer allows the same infrastructure primitive to serve both purposes without conflating them.

---

## 3. Data Boundaries

Each service exclusively owns its data. No other service may read or write that data except via the owning service's versioned REST API.

- Storage format is an implementation detail of the owning service.
- Storage must support tombstones, reference-based deletion ordering, and optimistic locking.

> **Rationale:** If two services share a database or read each other's storage directly, they become coupled at the data layer — schema changes in one break the other, deployment order matters, and the boundary between their responsibilities blurs. Enforcing data boundaries through versioned APIs is the only way to keep services independently deployable and independently understandable.

> **Rule:** Never access another service's CRDs or storage directly. Any code that does so is a defect.

### 3.1 Authority and Derived State

Every fact the platform acts on has exactly one **source of truth** — the system that authoritatively owns that fact. Every other representation of it is **derived state**: a cache, a mirrored record, a projected status field, an event payload. Derived state exists to make observation and coordination cheap; it is best-effort and may be stale, lagging, or entirely absent at the moment it is read.

The source of truth differs by fact, but is always identifiable:

| Fact | Source of truth | Derived (never authoritative) |
|---|---|---|
| A service's own resources | The owning service's storage, via its API (§3) | Another service's local copy or cache |
| A cloud-provider resource's existence | The provider, re-discovered by stable name or tag (§2.3) | The resource identifiers mirrored into CRD status |
| A strongly-consistent allocation | The allocator record keyed by a stable identity (§7.6) | Any copy of that allocation recorded elsewhere |
| A dependency's lifecycle state | The owning service, surfaced via status propagation (§5.8) | A controller's own last-reconciled snapshot |

> **Rule:** Derived state must never be the authority for a decision whose correctness depends on the real state — above all a destructive or irreversible one (deleting a resource, freeing an allocation, releasing a reference). Such a decision is made from the source of truth, re-read at the point of action. Derived state may be used to *optimise* (skip obviously-unnecessary work) but never to *authorise* skipping a step whose omission leaks or corrupts real state.

> **Rationale:** Projected status is written best-effort, after the fact, and a write can be lost to a crash, a resource-version conflict, or a reconcile that re-derives it from scratch. The absence of a record is therefore not evidence of the absence of the thing it would record. A cleanup step gated on "did we record this?" silently degrades to a no-op whenever that write was lost — and leaks the underlying resource. The authoritative system already knows the truth; ask it rather than trusting a mirror of it. See [§8.10](#810-authoritative-state-for-deprovisioning) for the deprovisioning rule this implies, and [Appendix A.7](#a7-best-effort-status-used-as-deprovision-authority) for the incident class it prevents.

---

## 4. Identity and Access

### 4.1 Organisational Hierarchy

The top-level organisational unit is the Organisation. Organisations contain one or more Projects. All user-managed resources are owned by a Project.

The hierarchy is: **Organisation → Project → Resources**. This is not merely a naming convention — it is the structural spine of the access model. Permissions granted at a higher level flow down to all nodes below.

An Organisation is the boundary of tenancy. Resources in one Organisation are never visible to principals of another Organisation unless explicitly shared.

### 4.2 Actors and Representation

Four distinct classes of actor interact with the platform:

| Actor | Representation |
|---|---|
| End User | A human operator authenticated via OIDC. Represented by a bearer token. Identity is established by token introspection at the public API boundary. |
| Service Account | An organisation-bound non-human identity managed within the Identity service. Authenticated via a bearer token issued by the Identity service. Authority is derived from group membership within the organisation, exactly as for end users. |
| System Account | A platform service acting autonomously. Represented by an X.509 certificate whose Common Name maps to an RBAC role. All service-to-service communication is exclusively via mTLS — the certificate is the identity. |
| Proxy Service | A platform service acting on behalf of an end user or service account. Carries both its own system account identity (mTLS) and the originating user's principal (propagated in context). See [section 4.4](#44-principals-and-proxies). |

### 4.3 RBAC

RBAC roles are composed of endpoint scopes, each with a CRUD permission set. Roles may be scoped at three levels: global, organisation, or project.

Roles are not assigned directly to users or service accounts. Group membership is the sole route to permissions: groups hold role assignments, and a principal's effective authority is the union of all roles held by all groups they belong to. A caller may only assign roles that contain permissions the caller already holds — privilege escalation through role assignment is not permitted.

The identity service exposes an ACL endpoint that returns the union of all role scopes for a principal. Every service enforces RBAC against this data — there is no local policy evaluation. Service code must not call this endpoint directly; it is accessed exclusively through the RBAC middleware library (`identity/pkg/rbac`, import path `github.com/unikorn-cloud/identity/pkg/rbac`), which handles token introspection, ACL retrieval, and scope enforcement as a unit. From the perspective of a handler author, RBAC is opaque: call the correct library check at the correct point in the handler and the middleware provides the rest.

Protected roles (`protected: true`) are not visible via the public API and may only be granted via Helm values at deployment time. These roles gate privileged platform operations.

Every public API endpoint requires authentication. The only unauthenticated endpoints are OIDC discovery and login flows.

**System account RBAC role registration.** When a new service is introduced, its system account certificate CN must be mapped to an RBAC role. This mapping is configured at deployment time via Helm values in the Identity service — it is not self-configured by the service. The role must be scoped to the minimum permissions required: declare only the `(resource, operation)` pairs the service actually exercises. Over-permissioned system account roles are a security defect. The CN-to-role mapping is the sole mechanism by which a service acquires its effective permissions; no other route exists.

### 4.4 Principals and Proxies

Four concerns govern how a principal relates to a resource. They are distinct and must never be conflated:

| Concern | Definition |
|---|---|
| **Attribution** | Who originated the work. The principal recorded on a resource at creation time. Determines quota and billing. Immutable. |
| **Placement** | Where the resource lives. The organisation and project that own the resource in the tenancy hierarchy. A proxy may provision resources in its own tenancy while attributing them to the user — attribution and placement can differ. |
| **Visibility** | Whether the principal can see the resource. Governed by RBAC scope. A principal may be the attributed owner of a resource without having the RBAC scope to list or read it. Attribution does not imply visibility. |
| **Effective authorization** | What the principal is permitted to do. Derived from the ACL at request time. Separate from both attribution and visibility. |

A **principal** is the actor identified by the attribution concern. The principal determines quota and billing. Once recorded on a resource it is immutable.

A **proxy** is a service provisioning resources on behalf of a principal. The proxy carries its own system account identity for transport authentication, but the principal it is acting for is propagated separately in the request context.

At the public API boundary, the caller's identity is established by token introspection (the actor claim). Organisational context — organisation and project — must be resolved at the earliest possible point and must be present before any downstream service call or resource write. In v1 services this context is present in the URL path. In v2 services there are two cases: for a root resource the caller must supply organisation and project in the request body; for a child resource (one whose tenancy is implied by a parent) the handler infers organisation and project by reading the parent resource's labels. See [section 7.10](#710-v2-api-design-model).

A proxy may provision resources in its own tenancy or in the principal's tenancy, depending on whether direct end-user access to those resources is appropriate.

### 4.5 Principal Propagation

- Every outbound service-to-service call must propagate the principal from the current request context.
- Every resource created must persist the principal as an immutable field.
- Every controller calling another service must propagate the principal from the resource under reconciliation — not from the service's own identity. Controllers operate outside the request/response cycle and have no live request context; the principal is reconstructed from the resource's persisted labels.
- Quota allocations must be made against the principal, never the proxy service.

#### 4.5.1 API Authentication Classifications

Every API endpoint falls into one of two visibility classifications, expressed via OpenAPI annotations. These annotations have no enforcement effect — access control is governed exclusively by RBAC policies:

**Public** — reachable by authenticated tenants via OAuth2 bearer token. The principal is derived from token introspection and RBAC is enforced against it. OpenAPI annotation: none (this is the default).

**Internal** — intended for service-to-service calls only. Suppressed from public API documentation via `x-hidden: true`. Access is restricted by RBAC policy (services authenticate via mTLS; only roles granted to system account CNs have access). No schema-level enforcement exists — `x-hidden: true` is a documentation marker, not a security control.

For internal endpoints, handler code determines how to use the caller's identity: against the mTLS CN alone (service principal, no user context), or against a user principal propagated in `X-Principal` (delegated principal). See §4.6 for the full principal handling rules.

### 4.6 Principal Propagation Modes and Impersonation

A user principal (`X-Principal`) may be propagated on any authenticated service-to-service call. Its presence alone does not determine how the receiving service uses it. Two explicit modes govern this, signalled by the presence or absence of `X-Impersonate`:

**Attribution-only propagation** — `X-Principal` is present; `X-Impersonate` is absent or false. The receiving service records the principal for billing, quota, and audit attribution only. It does not resolve RBAC against the propagated principal. Access decisions are made against the calling service's own mTLS certificate identity. This is the required mode for controllers provisioning resources on behalf of a user, where authorisation was established at the public API boundary and does not need to be re-evaluated on every downstream call.

**Impersonation** — `X-Principal` is present and `X-Impersonate: true` is also set. The receiving service treats the propagated principal as the authoritative actor for RBAC, quota, billing, and audit on that request. The effective ACL is the intersection of the user's ACL and the impersonating service's own system account ACL (see [§4.6.1](#461-acl-intersection-under-impersonation) below). This mode is only appropriate for proxy services presenting a user's identity downstream so that the full access control chain is preserved end-to-end.

**Rules:**

- `X-Principal` may be included on any authenticated service-to-service call for billing, quota, and audit attribution. Its presence alone does not trigger RBAC impersonation.
- RBAC impersonation is activated only when `X-Impersonate: true` accompanies `X-Principal`.
- In attribution-only mode, the receiving service resolves RBAC against the calling service's mTLS certificate identity.
- Impersonation (`X-Impersonate: true`) is only appropriate for proxy services. It is not a general-purpose capability.
- A service must never impersonate another service's identity. The mTLS certificate CN is fixed. Only the user principal context may travel in `X-Principal`.
- Every registered system service may signal impersonation — the impersonation gate is the same CN→role registration described in §4.3; no separate allowlist exists. However, impersonation only activates when a valid user principal is present in the request context (`X-Principal` header with a non-empty actor). Without a principal to synthesise RBAC against, the `X-Impersonate` header is ignored and the service's own mTLS identity is used for access decisions.

#### 4.6.1 ACL Intersection under Impersonation

When a system account (mTLS-authenticated service) impersonates an end-user, returning the user's ACL in full creates a confused deputy vulnerability: a user may hold permissions that the impersonating service itself is not authorised to exercise, and a narrowly-scoped proxy service could inadvertently acquire the user's broader administrative rights against the downstream service.

The effective ACL for an impersonated request is therefore the intersection of the user's ACL and the service's own system account ACL.

System accounts accumulate permissions exclusively at global scope. The service's global permissions act as a single allow-list that gates every scope level of the user's ACL. For each `(resource, operation)` tuple in the user's ACL at any scope level, the tuple is retained only if the service's global ACL also contains that `(resource, operation)`. If the service holds no permission for a resource type, all user permissions for that resource are stripped regardless of scope.

The scope hierarchy rule follows from the above:

| User's ACL scope | Service must hold |
|---|---|
| project | global (resource, op) |
| organization | global (resource, op) |
| global | global (resource, op) |

**Properties of this design:**

- **No privilege escalation** — impersonation can only reduce the user's effective permissions, never expand them.
- **Principle of least authority** — each service is confined to the subset of the user's permissions that the service itself is authorised for, regardless of the user's own role.
- **Attribution-only callers unaffected** — services that set `X-Principal` without `X-Impersonate` continue to resolve RBAC against their own system account role only, with no change in behaviour.

### 4.7 Federated Authentication

The Identity service does not maintain a user credential store. All human authentication is federated to upstream OIDC providers. Each organisation configures one or more upstream providers via the `OAuth2Provider` resource. When a user authenticates, they are redirected to their organisation's provider, which performs credential verification. Identity exchanges the upstream provider's token for a platform token normalised to the internal format.

The upstream provider's subject claim is the stable identity key. The same subject authenticating via the same provider is always the same platform user. A `User` resource records the global identity; an `OrganizationUser` resource records the membership and state (active, pending, suspended) within a specific organisation. A user may be a member of multiple organisations with independent states in each.

The `OAuth2Client` resource represents a client application — such as the platform UI — that is permitted to initiate the OAuth2 authorisation code flow. Clients are organisation-scoped; a client registered in one organisation cannot be used to authenticate against another.

> **Rationale:** Federation externalises credential management and MFA to the organisation's chosen identity provider, which is already trusted and likely already managing the user's other credentials. The platform does not need to solve password policies, MFA, or breach response — those are delegated. The platform only needs to trust the upstream assertion.

### 4.8 Token Infrastructure

The Identity service is the platform's OAuth2/OIDC authorization server. It does not merely validate tokens issued elsewhere — it issues all tokens used within the platform and is the trust root for all bearer-token-based authentication.

**Token classes.** The Identity service issues distinct token types for two bearer-token actor classes: federated user tokens (derived from upstream OIDC providers) and service account tokens (for org-bound non-human identities). Each class is normalized to the platform's internal format. Upstream provider tokens are never forwarded directly. System accounts authenticate exclusively via mTLS — they receive no bearer token from Identity; the certificate is the identity.

**Signing key rotation.** Tokens are signed using ES512. The Identity service maintains a rolling two-key window: the current primary key and the immediately previous key. New tokens are always signed with the current primary key. Verification accepts either key, meaning tokens survive one rotation before they become invalid. This window is the platform's guarantee against service disruption during key rotation. Any service that attempts to validate tokens locally must respect this two-key window; validation against a single fixed key is a defect.

**Token exchange.** The platform supports RFC 8693 token exchange for service-to-service trust chains. A caller presents a validated platform access token as a subject token; the Identity service issues a short-lived signed passport JWT recording the source identity, account type, organisation, optional project, and requested audience. This passport is the mechanism by which a service can prove delegated authority to a downstream service without forwarding the caller's full access token. Downstream services resolve RBAC normally against the Identity ACL endpoint — the passport does not embed permissions.

**Session model.** Each user has at most one active session per OAuth2 client. Refresh token rotation invalidates the prior token. This is the platform's primary defence against refresh token replay; a reused refresh token signals a compromised session.

**Protected protocol state.** Login dialog state, upstream OIDC state, and authorization codes are expressed as encrypted JWE tokens rather than persisted server-side state. This keeps the Identity service stateless with respect to in-flight auth flows.

---

## 5. Resource Graph

### 5.1 Edge Properties

There are four primitive edge properties. Every edge between two resource types carries some combination of them:

| Property | Description |
|---|---|
| Scope Propagation | Access granted at the source node flows to the target node. RBAC resolution traverses edges carrying this property. A principal's grant at any node covers all nodes reachable via scope-propagating edges below it. |
| Deletion Propagation (Forward) | Deletion of the source triggers deletion of the target. The source drives the target's deletion as part of its own teardown. |
| Deletion Propagation (Reverse) | Deletion of the target triggers deletion of the source. The source's existence is predicated on the target — if the target is removed, the source must be torn down. |
| Status Propagation (Upward) | A state change on the target enqueues the source for reconciliation. The source re-derives its own status from the aggregate state of all targets connected via this property. |
| Co-location Required | Source and target must be reachable from the same organisational scope root. Guaranteed implicitly where scope propagation is present; must be enforced explicitly otherwise. |

Deletion blocking is not a separate property — it is derived. Any edge carrying any deletion propagation property implies that the target blocks in DELETING state until the relationship is resolved.

### 5.2 Edge Types by Property Intersection

The three common relationship patterns in the platform are each a distinct intersection of these properties. Status propagation is orthogonal and may be carried by any edge type independently:

| Edge Type | Properties | Example |
|---|---|---|
| Containment | Scope Propagation + Reverse Deletion Propagation (blocking) + Forward Deletion Propagation (opt-in) + Status Propagation Upward (where applicable) + Co-location (implicit) | Organisation → Project; Cluster → Server |
| Consumption | Reverse Deletion Propagation (blocking) + Status Propagation Upward (where applicable) + Co-location Required | Server → Network |
| Dependency | Reverse Deletion Propagation (triggering) + Status Propagation Upward (where applicable) + Co-location Required | Cluster → Network |

A new resource type that does not fit these three patterns is expressed by declaring which primitive properties its edges carry — no new named types are required.

### 5.3 References

All edge relationships are expressed as named references on the target resource. A reference blocks deletion of the resource it is placed on. A resource in DELETING state blocks until all references are removed. References do not lapse automatically.

**Implementation mechanism.** References are implemented as Kubernetes finalizers on the referenced (target) resource. Placing a reference adds a finalizer to the target's `metadata.finalizers`; releasing a reference removes it. This means the Kubernetes API server enforces deletion blocking natively — a resource with outstanding finalizers cannot be garbage collected regardless of whether a controller is running. The reference API provides a service-boundary-safe interface for managing these finalizers across services.

References may cross service boundaries and are managed via the reference API. Reference operations are idempotent (both `PUT` to add and `DELETE` to remove return `200 OK`). The exact path structure is service-specific — v1 services include organisation and project in the path; v2 services address resources directly by UUID. Consult the target service's OpenAPI spec for the concrete endpoint.

The canonical reference string is derived by `GenerateResourceReference`: `{resource}.{group}/{uuid}`. This is stable for the lifetime of the target resource and unique across all resource types and groups. No other format is permitted for cross-resource references.

Adding a reference is idempotent — if already present the operation is a no-op. Removing a reference is idempotent — if absent the operation is a no-op. Both operations use optimistic locking; a write conflict is a transient condition and must be silently retried (see [section 8.2](#82-transient-conditions-and-silent-retry)).

References are placed by the consuming resource's controller and removed by the same controller. References are never placed or removed by the handler layer.

Two valid reference lifecycle patterns exist:

**Permanent hold** — the reference persists for the full lifetime of the consuming resource. Used when the dependency is only expressible at the Kubernetes level. Lifecycle: place reference → use resource → remove reference as part of deprovisioning → remove finalizer. The consuming resource's own finalizer must not be removed until all outbound permanent references have been released.

**Transient provisioning guard** — the reference is placed at the start of a Provision pass and removed once the downstream infrastructure has taken ownership of the live dependency. Used when the consumed resource is referenced by external infrastructure after provisioning completes, and the external system is the authoritative guard against deletion during normal operation. On subsequent reconcile passes, the guard is re-applied and released again around the provisioning call.

### 5.4 Event Bus

The event bus is a push-based messaging system through which services publish resource lifecycle events and subscribe to events from services they depend on. The abstraction is backend-agnostic: the current implementation uses Kubernetes list/watch, and the interface is designed so the backend can be replaced (e.g. with Kafka or NATS) without changing consumer code or event semantics.

- On service start or restart, all existing resources on the subscribed topic must be replayed. This allows subscribers to trigger or re-evaluate any deletion that was missed during downtime. All subsequent events are streamed.
- Event delivery is at-least-once. Subscribers must handle duplicate events idempotently. Exactly-once delivery is not guaranteed and must not be assumed.
- Event payloads are minimal: `resourceID` and optional `deletionTimestamp`. No resource state is embedded in the event. Subscribers must call the owning service's versioned API to retrieve current state. This preserves data boundaries — the event is a stimulus, not a data transfer.
- The `deletionTimestamp` field allows subscribers to filter the stream and distinguish deletion events from creation or update events without an additional API call.
- The event bus is the required mechanism for all cross-service deletion and status propagation. Intra-service propagation uses Kubernetes watches directly. The semantics are identical — only the plumbing differs.

> **Implementation note:** The current backend is Kubernetes list/watch (`core/pkg/messaging/kubernetes`). This is the correct and only approved mechanism for cross-service event consumption — do not implement ad-hoc watch loops, poll loops, or direct subscriptions to another service's storage. The abstraction is designed so the backend can be replaced (Kafka, NATS, etc.) without changing consumer code or event semantics.

### 5.5 Containment

A containment edge expresses that a resource cannot exist outside the scope of its container. The contained resource's existence is logically bounded by the container. Example: an Organisation contains a Project; a Project contains a Server (via transitive closure).

The "cannot exist outside" invariant is enforced by the reverse deletion propagation (blocking) property that containment edges carry by default: contained resources place blocking references on their container, so the container cannot be deleted while any contained resource exists. Forward deletion propagation (opt-in) is the automation that cascades deletion from container to contained resources. Without it, contained resources must be manually deleted before the container can be removed — the blocking references prevent the container from disappearing first.

- Containment is transitive. A resource reachable from a given node via a chain of containment edges is part of that node's subtree, regardless of how many hops separate them.
- Containment edges span service boundaries. No single service owns the full containment graph.
- List operations are always filtered to the intersection of the principal's ACL and the requested scope, traversing containment edges. A principal never receives resources outside their permitted subtree.

### 5.6 Consumption

A consumption edge expresses that a resource uses another resource it does not contain. The consumed resource has independent existence and may be shared by multiple consumers. Example: a Server consuming a SecurityGroup.

- A consumer must place a reference on the consumed resource before use.
- Both nodes must lie within the same subgraph rooted at the Organisation — the co-location constraint.
- Standalone deletion of a consumed resource with active references is prohibited.

### 5.7 Dependency

A dependency edge expresses that a resource's existence is predicated on another resource. If the depended-on resource is deleted, the dependent must be deleted too. Example: a Cluster whose Servers are placed on a Network — deleting the Network triggers deletion of the Cluster.

- A dependency edge carries reverse deletion propagation with active force: it is a trigger, not a block. The dependent's controller must watch for the target entering DELETING state and initiate its own deletion in response. The target deletes freely — it is not blocked by the dependent.
- This relationship is strictly asymmetric. Deletion of the depended-on resource triggers deletion of the dependent. Deletion of the dependent never propagates to the depended-on resource.
- Because deletion may arrive via the dependency path rather than the expected top-down containment path, controllers must treat the absence of contained resources as a valid state and handle it idempotently.
- Both nodes must satisfy the co-location constraint.

### 5.8 Status Propagation

Status propagation is orthogonal to deletion semantics. Where an edge carries status propagation upward, a state change on the target must enqueue the source for reconciliation. The source then re-derives its own status from the aggregate state of all connected targets.

- A controller implementing status propagation upward must register an explicit watch on the target resource type. It is the watch — not a periodic poll — that converts a state change on the target into a work queue entry on the source.
- For intra-service edges, Kubernetes watches on the target resource type satisfy this directly.
- For cross-service edges, the event bus provides the stimulus.
- Status propagation upward may be carried by any edge type. Whether a given edge carries it is a declaration made when the resource type is defined. It is not assumed.

### 5.9 Deletion Propagation Mechanisms

**Intra-service edges** — Kubernetes owner references are the preferred mechanism. They natively implement the required containment and deletion ordering semantics. Setting an owner reference at create time is sufficient; Kubernetes garbage collection handles propagation automatically.

**Cross-service edges** — the event bus is the required mechanism (see [section 5.4](#54-event-bus)). When a resource is tombstoned a deletion event is published. Subscribing services react and initiate deletion of their dependent or consuming resources. Correctness is guaranteed by the reference blocking mechanism — a resource cannot complete deletion until all references are released, regardless of how the stimulus arrived.

### 5.10 Defining a New Resource Type

Adding a new resource type to the graph is a formal exercise. For each relationship the new type has with an existing type, declare which primitive properties the edge carries. The controller implementation follows directly:

| Property | Required implementation |
|---|---|
| Scope Propagation | Implement RBAC filtering that traverses this edge. The principal's ACL must include this resource type within the appropriate scope. |
| Forward Deletion Propagation | The controller must initiate deletion of the target when the source is deleted, and must yield until the target reaches DELETED. |
| Reverse Deletion Propagation (blocking) | The controller must place a reference on the target at creation and release it before the source is fully deleted. |
| Reverse Deletion Propagation (triggering) | The controller must watch the target for a deletion timestamp and initiate its own deletion in response. No blocking reference is placed on the target. |
| Status Propagation Upward | The controller must register an explicit watch (intra-service) or event bus subscription (cross-service) on the target. On any state change, re-derive source status from the aggregate state of all connected targets. |
| Co-location Required | The handler layer must validate at create time that source and target share the same organisational scope root. Reject with 422 if not. |

### 5.11 Deletion Semantics

- By default, deletion of a resource blocks until all references are released. The graph structure does not imply cascading deletion.
- Cascading deletion — where deletion propagates down containment edges — must be explicitly opted into via the forward deletion propagation property. It is not the default and must be documented by the service that implements it.
- Cascading must never propagate across consumption or dependency edges to resources outside the subtree.
- Inter-service deletion ordering uses Kubernetes finalizers only. Spec fields must not be used — spec is immutable once a deletion timestamp is set.
- Intra-service ordering may use owner references and blocking deletes.
- Each deprovision step is driven from the source of truth and is idempotent. A step must never be gated on best-effort recorded status — absence of a status record is not evidence the underlying resource is absent (see [§3.1](#31-authority-and-derived-state), [§8.10](#810-authoritative-state-for-deprovisioning)).

### 5.12 Deletion State Machine

Deletion proceeds through three internal reconciler phases. These are not distinct API-visible condition states — the API-visible condition reason throughout DELETING, DRAINING, and FINALIZING is `ConditionReasonDeprovisioning`. `ConditionReasonDeprovisioned` is set immediately before the controller's own finalizer is removed — as the last status write the controller will ever make to the resource. Kubernetes garbage collects the object after the finalizer is gone.

| Phase | Description |
|---|---|
| DELETING | `DeletionTimestamp` set on the Kubernetes object. Deletion event published. Reconciler blocks while any inbound reference finalizers exist (see [§5.3](#53-references)). |
| DRAINING | All reference finalizers cleared — every holder has released its claim. Reconciler cleans up contained child resources. |
| FINALIZING | Contained children gone. Reconciler releases outbound references held on consumed or depended-on resources, then removes its own finalizer. |
| DELETED | Reconciler sets `ConditionReasonDeprovisioned`, then removes its own finalizer as the last action. Kubernetes garbage collects the object after the finalizer is gone. |

### 5.13 Deletion Events

- Deletion events must be handled idempotently. Duplicate events are expected.
- Event payloads carry only `resourceID` and `deletionTimestamp`. No resource state is embedded. Subscribers must call the owning service's API to get current state.

---

## 6. Resource Model

### 6.1 Naming and Metadata

- Resource identifiers are UUIDs, globally unique and immutable for the lifetime of the resource. A resource ID is never reused, reassigned, or modified. Two strategies exist — the choice is architectural, not cosmetic (see [Appendix A.1](#a1-toctou-race--resource-name-uniqueness)):
  - **UUID v4 (random)** — default for all resources. Use `GenerateResourceID` / `NewObjectMetadata`. The platform is the sole authority on the name; no natural key exists. Each lifetime of the resource gets a distinct UUID, keeping billing and audit records unambiguous across recreations.
  - **UUID v5 (deterministic)** — use when the resource participates in a uniqueness contract derived from immutable content fields (e.g. network-id + hostname). Use `GenerateDeterministicResourceID` / `NewDeterministicObjectMetadata`. Two creates with the same natural key produce the same UUID and collide at the storage layer, giving native 409 conflict detection without a read-before-write. Trade-off: if the resource is deleted and recreated with the same natural key, it gets the same UUID — billing and audit records from a previous lifetime share the identifier with the new resource.
- Human-readable names are stored in `metadata.labels[<platform-label-prefix>/name]` and are mutable. Never hardcode resource names.
- Resource status is expressed exclusively via `status.conditions`, following the `metav1.Condition` schema with strongly-typed reason constants. There is no authoritative phase field — conditions are the sole source of truth for resource state.
- Operations that require a controller to drive the resource to its terminal state are asynchronous and return `202 Accepted` immediately. Operations that are fully effected within the request/response cycle are synchronous: reads return `200 OK`, creates return `201 Created`. Always read `status.conditions` before acting on resource state. See [§7.1](#71-synchronous-vs-asynchronous-operations) for the normative definition.

### 6.2 Labels and Annotations

Labels are not decoration — they are the platform's query model. List operations, scope resolution, RBAC filtering, and resource ancestry traversal all rely on labels being present and correct. A resource with missing labels is not merely untidy; it is invisible to the operations that depend on those labels and may silently escape security filtering.

Labels fall into three functional categories:

| Category | Purpose | Examples |
|---|---|---|
| Scope | Encode the tenancy scope of the resource. Used by list operations and RBAC filtering. | organisation ID, project ID |
| Ancestry | Encode the resource's position in the dependency graph. Required for multi-hop queries and co-location validation. | region ID, network ID, identity ID |
| Attribution | Record who created the resource and on whose behalf. Used for billing, quota, and audit. | creator subject, principal organisation, principal project |

All label keys use the platform label prefix defined in the core library constants. Services must not invent ad-hoc label keys. The full set of platform labels and their semantics is defined in `core/pkg/constants`.

**Ownership rules:**
- Labels and annotations are exclusively owned by the server handler layer via `conversion.NewObjectMetadata` (create) and `conversion.UpdateObjectMetadata` (update).
- No controller, external process, or agent may add, modify, or remove them. Any externally-written values are silently overwritten on the next write.
- Changes to label or annotation semantics require a revision to this specification before implementation.

### 6.3 Tags

Tags are user-defined key-value pairs attached to resources via `spec.tags`. They are distinct from labels: they are user-visible, user-controlled, and have no effect on RBAC, scoping, or query behaviour.

Tags are used for user-defined resource organisation — cost attribution, environment markers, team ownership. They are applied as a post-list filter in-process after storage retrieval; they are not expressed as storage-level selectors. This means tag filtering does not reduce the storage query cost and cannot be used as a substitute for scope labels.

**Platform-reserved tag keys.** The platform currently uses reserved tag keys within service-owned namespaces (e.g. `compute.unikorn-cloud.org:instance-id`) for internal resource linkage — for example, to recover the backing `region.Server` for a `ComputeInstance` without storing an explicit foreign key. These reserved keys are managed exclusively by the platform, must not be set by users or agents, and are stripped from user-supplied payloads before being re-applied by the platform.

**Intended direction.** The use of tags for internal linkage is a transitional pattern. `spec.tags` should converge to being exclusively user-facing: values that are opaque to the platform and used only for user-defined resource organisation. Internal resource linkage should move to explicit labels (§6.2) or stored spec fields. New services should not use tags for internal resource coordination — prefer labels or explicit spec fields.

---

## 7. API Design

API handlers operate over Kubernetes objects without transactional guarantees. Multi-object handler operations are not atomic: a failure mid-sequence leaves partial state that may require controller-driven or idempotent-retry cleanup. Sagas ([§7.5](#75-multi-step-operations-sagas)) are the mitigation on create and update paths where partial completion must be rolled back on failure.

Controllers are idempotent and eventually consistent by design — this is the intended model, not a limitation. The requeue mechanism and idempotent reconcile loop handle partial progress naturally without requiring atomic operations.

### 7.1 Synchronous vs Asynchronous Operations

The distinction between synchronous and asynchronous operations reflects whether an operation can be fully effected within the request/response cycle, or whether it initiates work that requires a controller to drive to completion.

**Synchronous** operations are ones whose effect is immediate and complete by the time the response is returned. No further processing is required. Examples: reading a resource, applying or releasing a reference.

- Synchronous reads return `200 OK`. Synchronous creates return `201 Created`.

**Asynchronous** operations cannot be completed immediately — they involve provisioning, configuring, or tearing down real infrastructure or dependent resources. Every asynchronous operation must have an accompanying controller responsible for driving the resource to its terminal state.

- Long-running operations must never be performed synchronously.
- Asynchronous operations return `202 Accepted` immediately. The response body contains the resource with its initial transitional conditions set. The caller reads `status.conditions` to track progress.
- If a resource already has a deletion timestamp set, return `202` without re-triggering deletion. If the resource is not found, return `404`.

### 7.2 Handler Layer Responsibilities

**Prerequisites** (enforced in order before any business logic executes):

1. Introspect the bearer token against the identity service to establish the principal.
2. Retrieve the ACL for the principal and enforce the required scope and CRUD permission for the operation.
3. For **any user-supplied resource ID** (URL path parameter, query parameter, or request body field referencing a resource owned by this or another service): fetch the referenced resource, derive its tenancy scope (org/project) from its labels, and verify the caller's ACL (from step 2) permits the required operation at that scope. This is the same check applied on every read and list path. It is necessary in addition to step 2 because the resource's tenancy scope is not known until the resource is fetched — in v2 APIs the URL contains only a UUID, not org/project. No shortcuts: the check must be performed even if the resource type is already covered by step 2. See [section 10.1](#101-platform-security-invariants) — input-path authorization invariant.
4. Validate the request body against the OpenAPI schema (performed by middleware before the handler is invoked).

**General handler invariant:** The principal must be complete — including org/project — before any downstream service call or resource write. For v2 services this means completing the principal context from the request body or parent resource labels ([§4.4](#44-principals-and-proxies)) before any business logic proceeds.

**Create:** Derive labels and annotations via `conversion.NewObjectMetadata` from the resolved organisational context (URL path parameters for v1; request body or parent resource labels for v2) and the authenticated principal. Never copy labels or annotations verbatim from the request body. Add the controller's finalizer to the resource metadata before any controller is able to observe the resource.

**Update:** Read the current resource (`GetRaw`), re-derive labels and annotations via `conversion.UpdateObjectMetadata` using the current resource's own label values as inputs. Never copy labels or annotations verbatim from the request body. Patch with optimistic locking (`MergeFromWithOptimisticLock`).

**Delete:** Verify `DeletionTimestamp` is nil before issuing the delete. If already set, return `202` idempotently.

#### 7.2.1 Handler Authentication Classification

Every handler must declare its authentication classification via OpenAPI extensions on the operation or path. The classification must match [section 4.5.1](#451-api-authentication-classifications) exactly.

- **Public handlers** (no annotation): principal derived from OAuth2 bearer token via token introspection. The handler calls `rbac.AllowProjectScope` or `rbac.AllowOrganizationScope` against the derived principal.
- **Internal handlers** (`x-hidden: true`): access restricted by RBAC policy (mTLS service accounts). The annotation is documentation-only. The handler inspects the request at runtime: if `X-Principal` is absent, RBAC is resolved against the mTLS CN; if `X-Principal` is present, RBAC is resolved against the propagated user principal. See §4.6 for principal handling modes.

#### 7.2.2 Server Middleware Stack

The canonical server setup defines two ordered middleware groups.

**Pre-routing middleware** (applied in this order):

1. **OpenTelemetry** — establishes a trace context. Must run first so the trace ID is available to all subsequent middleware and handlers.
2. **Logging** — captures structured request/response log entries. Must run after OpenTelemetry so log entries carry the trace ID.
3. **Route Resolver** — resolves the matched OpenAPI operation and injects it into the request context. Required by CORS, OpenAPI validator, and audit middleware.
4. **CORS** — synthesises OPTIONS responses and injects CORS headers. Must run pre-routing so preflight requests are handled before authentication is applied.

**Post-routing middleware** (registered in this order; applied in reverse — last registered is outermost):

1. **`audit.Middleware`** — registered first, therefore outermost. Wraps the full post-routing chain. Emits the audit log entry after the inner chain completes.
2. **`validator.Middleware`** — registered second, therefore innermost (runs first). Validates the request against the OpenAPI schema and performs token introspection and principal population. Authentication failure returns `401`/`403` before the handler is reached.

All platform services must replicate this middleware stack exactly. Deviations must be approved and documented here.

### 7.3 Conflict Detection

All write operations must include a conflict detection mechanism. Conflict detection is implemented by including the resource version in the write request (`MergeFromWithOptimisticLock`). If the resource version on the server has advanced since the client last read the resource, the write must be rejected with `409 Conflict`. The caller is then responsible for re-reading and re-applying their change.

**Rationale — why the server must not retry internally:**
Server-side retry after a conflict would silently overwrite a concurrent write. The server does not know whether the new incoming change is still valid against the updated resource state; only the caller does. Surfacing the 409 forces the caller to re-derive its intent from the current state, which is the only safe approach under concurrent writes.

**OpenAPI requirement:**
Whether a `409` must be advertised depends on whether the operation can actually produce a conflict:

- **All PUT (update) endpoints** must include `409`. Section 7.2 mandates `MergeFromWithOptimisticLock` for every update; since Kubernetes will raise `IsConflict` whenever the resource version has advanced, a conflict is always a possible outcome regardless of traffic patterns.
- **POST endpoints that patch an existing resource** (e.g. rotate operations that call `MergeFromWithOptimisticLock`) must include `409` for the same reason.
- **POST endpoints that create a new resource** should include `409` if the handler enforces name or uniqueness constraints that can produce a duplicate (the "already exists" flavour of conflict). If no such check exists, omitting `409` is acceptable and more accurate.

All `409` responses must reference the canonical `conflictResponse` component from the core spec.

**Client-side contract:**

| Caller type | Required behaviour |
|---|---|
| Controller (internal) | Treat as transient. Return `ErrYield` to requeue with a fixed timeout. Re-read the resource before the next reconcile pass. See [section 8.7](#87-downstream-error-handling). |
| Service-to-service handler | Re-read the target resource and re-apply the change within the same request. If a second conflict occurs, return `409` to the upstream caller rather than looping. |
| UI / interactive client | Do not auto-retry silently. Inform the user that their change conflicted with a concurrent modification and prompt them to reload before retrying. Auto-retry risks overwriting a change the user has not seen. |
| SDK / programmatic client | Re-read the resource, re-derive the change from the fresh state, and retry once. If the conflict recurs, surface the error to the caller. |

### 7.4 Resource References

Resource references are defined in [section 5.3](#53-references). The canonical reference string format, idempotency guarantees, implementation via `GenerateResourceReference`, and both reference lifecycle patterns (permanent hold and transient provisioning guard) are all specified there.

### 7.5 Multi-Step Operations (Sagas)

The saga pattern from the core library must be used for any handler operation that produces side effects across multiple systems within the synchronous request/response cycle.

- Each saga action must have a corresponding compensating transaction. If any action fails, completed actions are rolled back in reverse order before the error is returned to the caller.
- The terminal action — the resource create or patch that hands off to a controller — has no compensating transaction. Once the resource exists, cleanup is the controller's responsibility via the deletion path.
- Sagas must only be used for side effects that are fully reversible within the handler's synchronous scope. Any side effect that produces persistent state requiring a controller to clean up must not be modelled as a saga step — it must be modelled as a controller operation with idempotent allocation and deallocation.

### 7.6 Quota and Strongly Consistent Allocations

The platform quota system tracks consumption of arbitrary, service-defined resource kinds. Quota limits are set at the organisation level. A new service that introduces a user-accountable resource type must declare a new quota kind in the Identity service's quota schema and configure a default limit at deployment time via Helm values. The quota kind string is the canonical name used in all allocation and reservation operations for that resource type — it must be stable for the lifetime of the service.

Quota allocation uses a two-phase reservation model. The handler makes a soft reservation synchronously. The controller promotes the reservation to a committed allocation and owns the full allocation lifecycle.

- **On resource create**: the handler computes the reservation key via `GenerateResourceReference` and checks whether available quota (limit minus committed allocations minus active reservations) is sufficient. If insufficient, the handler returns `403 Forbidden` immediately. If sufficient, the handler creates a soft reservation keyed on the resource UUID before returning `202`. The reservation reduces the available balance immediately, preventing concurrent requests from racing past the quota limit.
- **The controller** promotes the reservation to a committed allocation on the provisioning path. Promotion is idempotent on the key.
- **The controller** releases the reservation or committed allocation (whichever is current) on the deprovisioning path, before removing its finalizer. Release is idempotent on the key.
- The handler adds the controller's finalizer to the resource at creation time, before returning `202` (see [section 7.2](#72-handler-layer-responsibilities)). This guarantees the deprovisioning path always runs before the resource is garbage collected, even if deletion is requested before the controller's first reconcile. Any external resource, reservation, or side effect created during handler execution — quota reservations, cross-service resources, index entries — is cleaned up unconditionally by the deprovisioning path. No special handling is required for deletion arriving before the controller has run.
- The quota system must treat all allocation, promotion, and release operations as idempotent on the key.
- The quota service must provide atomic check-and-reserve. The available balance check and the reservation creation must be a single atomic operation.

### 7.7 API Versioning

- Public-facing APIs must be backward compatible within a version. Additive changes — new optional fields, new endpoints, new enum values — are permitted without a version bump. Removing fields, changing field semantics, or altering response structure requires a version bump.
- A deprecated version must remain available for a defined deprecation period before removal. Deprecation must be communicated via API response headers and documented in the changelog.
- Inter-service APIs may be broken deliberately, provided the producing service and all consuming services are updated in the same coordinated release. No deprecation period is required.
- Major version bumps — paradigm shifts rather than incremental changes — must be coordinated across all services simultaneously. Partial major version rollouts that leave consumers on the old version are not supported. The v2 release is the canonical example of a coordinated major bump.

### 7.8 List Filtering

> **Note:** This section describes the v1 list filtering model, retained for reference only. The v2 model ([section 7.10](#710-v2-api-design-model)) supersedes it for all new services. Do not design new list operations against the v1 model.

- List operations filter by organisation and project via label selectors derived from the principal's ACL.
- Post-list RBAC filtering (`rbac.AllowProjectScope`) is applied in-process where label selectors alone cannot encode the access policy.
- Tag-based filtering (`spec.tags`) is applied after the list, not as a label selector.
- List results are sorted stably (by name) to ensure deterministic ordering.

### 7.9 Error Handling and Propagation

All API errors are represented as a typed error carrying an HTTP status code, a machine-readable error code, and a human-readable description. Internal context is attached for logging only and is never included in the response body.

Every error response body carries: `error` (machine-readable code), `error_description` (human-readable description written for the authenticated legitimate user), and optionally `trace_id` (OpenTelemetry trace ID for log correlation).

**Standard error codes:**

| HTTP Status | Code | Meaning |
|---|---|---|
| 400 | `invalid_request` | The request was syntactically malformed or violated a protocol constraint. |
| 401 | `access_denied` | Authentication failed or the token has expired. Response includes a `WWW-Authenticate` header pointing to the OIDC protected resource metadata endpoint (RFC9728). |
| 403 | `forbidden` | The principal is authenticated but does not have permission to perform the requested operation. |
| 404 | `not_found` | The requested resource does not exist. |
| 405 | `method_not_allowed` | The HTTP method is not supported on this endpoint. |
| 409 | `conflict` | The resource already exists or a write conflict was detected. The caller must re-read before retrying. |
| 413 | `request_entity_too_large` | The request body exceeds internal size limits. |
| 422 | `unprocessable_content` | The request is syntactically valid but semantically incorrect — e.g. a co-location constraint violation or an invalid field combination. |
| 500 | `server_error` | An unexpected internal error occurred. No detail is included in the response. The trace ID is included to correlate with server-side logs. |

**Security rules:**

- Error descriptions must contain only what the legitimate user needs to take corrective action. They must not contain implementation details, infrastructure topology, library or framework identities, file paths, or internal hostnames. (OWASP API Security Top 10 API8:2023, CWE-209)
- Where revealing that a resource exists would constitute a data boundary violation, or where the resource has scoped visibility (ACL-restricted, impersonation-filtered), return `404` uniformly regardless of whether the actual cause is non-existence or forbidden access. Never use `403` as an existence oracle. Phrasing such as "resource does not exist or is not accessible" is equally prohibited — distinguishing the two cases leaks existence information. (OWASP WSTG-ERRH-01, CWE-208)
- Authentication error responses must not distinguish between an unknown principal and an incorrect credential. Both return `401 access_denied` with a non-specific description. (OWASP WSTG-ERRH-02)
- Error descriptions must never reference resources, identifiers, or state belonging to a scope the requesting principal cannot access.
- When a handler calls a downstream service and receives an error response, it must propagate the typed error faithfully — with one exception: if the downstream returns `403` for a resource that would constitute an existence-oracle violation at this service's data boundary (i.e. the requesting user should not know the resource exists), map the `403` to `404`. Internal service-to-service `403` responses caused by mTLS misconfiguration propagate as `403` since they are not existence-oracle issues.
- Untyped errors are caught by the terminal error handler and written as `500 server_error` with no detail.
- The `401` response includes a `WWW-Authenticate` header per RFC6750 and RFC9728. Services must not construct this header manually — use the canonical `AccessDenied` constructor from the core library.

### 7.10 v2 API Design Model

The v2 API is the target design for all platform services. New services must follow the v2 model. The v1 model is retained only for backward compatibility in existing services and must not be used as a reference for new work.

**Flat routing.** v1 APIs embedded organisation and project in the URL path, coupling the routing structure to the tenancy hierarchy. This made cross-project resource references awkward and produced deeply nested paths. v2 routes resources directly by their UUID. Organisation and project context is not required in the URL — it is inferred from the resource itself, from a field in the request body, or from a related resource resolved during the request.

**Relationship-driven scoping.** In v1, list operations traversed the URL hierarchy to determine scope. In v2, list operations are bounded by the principal's ACL and optionally narrowed further by caller-supplied query parameters (e.g. organisation ID, project ID, region ID). The server constructs a label selector from the ACL and any supplied query parameters, then applies per-item RBAC filtering. Scope is a property of the resource, expressed through its labels, not a property of the path.

**Direct resource addressing.** Resources are addressed by their UUID in all operations. There is no hierarchical path traversal to reach a resource. A handler resolves scope by reading the resource and walking its label-encoded ancestry, not by interpreting the URL structure.

**Principal completion.** Because organisation and project are no longer present in the URL, write-path handlers must complete the principal context before writing audit, quota, or attribution fields or making any downstream call. The mechanism depends on the resource type: for a root resource, the caller supplies organisation and project in the request body; for a child resource, the handler reads the parent resource and infers organisation and project from its labels. In both cases completion must happen before any downstream call is made.

> **Rationale:** Hierarchical URL routing encodes tenancy structure into the API surface. As the platform's resource graph grows more complex — resources owned by one project but consumed by another, cross-region references, platform-managed resources attributed to an org — a hierarchy-in-URL model cannot express these relationships without becoming incoherent. Flat routing with label-encoded scope decouples the API surface from the tenancy model and allows the resource graph to evolve independently.

### 7.11 OpenAPI-First Development

The OpenAPI specification is the source of truth for every service API. All server stubs, client SDKs, request validation, and middleware routing are generated from it. The implementation must conform to the spec — the spec is never updated retroactively to match the implementation.

**When building a new service, the OpenAPI spec is the first artefact to write.** Before any Go code is authored:

1. Define all resource types, request/response schemas, and endpoint paths in the OpenAPI YAML.
2. Annotate each operation with the correct visibility classification: no annotation (public) or `x-hidden: true` (internal). For internal endpoints that expect a propagated user principal, document this in the endpoint description — it is a handler-code contract, not a middleware concern.
3. Declare all `4xx` and `5xx` responses the operation can produce. The OpenAPI validator will reject responses that are not declared.
4. Run code generation. Server stubs, client types, and the validator configuration are all derived from the spec.

**OpenAPI annotations:**

| Annotation | Effect |
|---|---|
| *(none)* | Public endpoint. Visible in public API documentation. |
| `x-hidden: true` | Internal endpoint. Suppressed from public documentation (Mintlify). No enforcement effect — access is restricted by RBAC policy, not by the annotation. |

---

## 8. Controller Behaviour

Each controller is its own binary. `pkg/manager.Run` accepts exactly one `ControllerFactory` and manages exactly one controller type per process invocation. A service that manages N resource types ships N controller binaries, each with its own `main.go` calling `manager.Run` with its own factory.

Each controller manages exactly one resource type. The reconciler is a function that takes a resource and drives it from its current observed state toward its desired state. The manager calls the reconciler when a watch event or a requeue fires.

### 8.1 The Work Queue

The work queue decouples event production from event processing. Events are produced by resource watches and by controllers explicitly requeuing themselves. The queue deduplicates — if a resource is enqueued multiple times before it is processed, it is processed once.

Processing is the reconcile function. It runs to completion and then the item is removed from the queue. There is no streaming, no callback, no blocking wait inside the function — if the controller cannot proceed, it returns and puts itself back on the queue.

- Reconciliation must be idempotent.
- If a resource has been paused, the controller must skip reconciliation entirely and return immediately without requeuing. Pause is an operator escape hatch, set via annotation, that halts all reconciliation on a resource for maintenance or debugging. A paused resource remains in whatever condition it was in when paused.

### 8.2 Transient Conditions and Silent Retry

A transient condition is any condition that is expected to resolve without intervention and will not produce the same outcome on retry. Controllers must never surface transient conditions to the user as failures. The resource remains in its current in-progress condition, the reconcile returns, and the controller retries after a fixed interval.

Transient conditions include, but are not limited to: a dependency not yet provisioned; references not yet released; a remote resource not yet in the required state; a write conflict (409); a temporary downstream service unavailability (5xx); an event bus stimulus arriving before the resource state is queryable.

- A transient condition must never set the resource's condition to `Errored`.
- The retry interval is fixed and constant. It must not grow.
- `ErrYield` is the implementation mechanism. Returning `ErrYield` from the reconcile function signals the controller runtime to requeue the item after `DefaultYieldTimeout` without treating it as a failure.

### 8.3 Genuine Errors

A genuine error is an unexpected condition that cannot be resolved by requeueing. Examples: a malformed resource that cannot be processed, an unrecoverable failure in a downstream system, an assertion that should never be false.

- A genuine error sets the resource's `Available` condition to `ConditionReasonErrored` and writes the error message to `status.conditions`. This is the correct and only channel for surfacing failures the user needs to act on.
- Genuine errors must be reserved strictly for conditions that are truly unrecoverable without intervention.
- **Provisioning path:** genuine errors are returned as `reconcile.Result{RequeueAfter: DefaultYieldTimeout}, nil` — a fixed-interval requeue, not an error return. This is a deliberate design choice: returning an error triggers controller-runtime's exponential backoff, which produces slow recovery during transient infrastructure instability and a poor UX (users see a growing gap between retries). Fixed-interval requeue keeps recovery fast and visible via `ConditionReasonErrored`.
- **Deprovisioning path:** genuine errors are returned as real errors (`reconcile.Result{}, err`). The controller-runtime applies exponential backoff. Deprovisioning failures are more serious consistency problems; the slower retry is acceptable and the backoff reduces noise.

### 8.4 Status Conditions

Every pass through the reconcile function, successful or not, must update the resource's `Available` condition before returning. Two legitimate exceptions exist where the reconciler exits without writing status:

- **Resource already deleted** — `DeletionTimestamp` is set but no finalizers remain; the resource is awaiting Kubernetes GC. There is nothing to write to.
- **Resource is paused** — the `Paused()` annotation is set. The controller exits immediately; the resource retains its last-written condition until unpaused.

In all other cases, a reconcile that exits without writing status has failed its contract with the queue.

| Outcome | Condition | Queue behaviour |
|---|---|---|
| Transient (`ErrYield`) | `ConditionReasonProvisioning` or `ConditionReasonDeprovisioning` | Requeued at fixed interval |
| Genuine error (provisioning) | `ConditionReasonErrored` | Requeued at fixed interval (see §8.3) |
| Genuine error (deprovisioning) | `ConditionReasonErrored` | Requeued with exponential backoff |
| Context cancelled (shutdown) | `ConditionReasonCancelled` | Requeued at fixed interval; condition clears on next reconcile |
| Success | `ConditionReasonProvisioned` or `ConditionReasonDeprovisioned` | Removed from queue |

If the status write itself fails (e.g. due to a resource version conflict), the controller must return `reconcile.Result{RequeueAfter: DefaultYieldTimeout}, nil` — not an error. A status write failure is a transient queue item; returning an error would trigger exponential backoff, which must not happen for infrastructure-level retries.

### 8.5 Finalizer Lifecycle

Every reconcile pass must check the deletion timestamp before taking any other action. The ordering is not optional:

1. **Check `DeletionTimestamp`** — if set, the resource is being deleted. Jump immediately to the deprovisioning path; skip all provisioning logic.
2. **Deprovisioning path** — run the deprovision sequence (release references, delete owned children, release quota allocation). When deprovisioning is complete, remove the finalizer. The resource is then eligible for garbage collection.
3. **If `DeletionTimestamp` is not set** — ensure the finalizer is present; add it unconditionally if absent. The handler adds it at creation time; the controller adds it here as an unconditional guarantee that covers any gap. Proceed to provisioning.
4. **Provisioning path** — with the finalizer confirmed present, proceed with provisioning.

Provisioning without first ensuring the finalizer is present is a defect. It creates a window in which deletion can bypass the deprovisioning path entirely.

### 8.6 Controller Watches

Controllers that implement status propagation upward must register watches on the resources whose state they depend on.

- For intra-service edges: the controller registers a Kubernetes watch on the target resource type.
- For cross-service edges: the controller subscribes to the event bus topic published by the owning service. On service start or restart, the event bus replays all existing resources — the controller must handle replayed events idempotently.
- Watch loss on service restart is handled by the event bus replay semantics. Controllers must not implement their own replay.

### 8.7 Downstream Error Handling

When a controller receives an error response from a downstream service call:

| Response | Treatment |
|---|---|
| 5xx | Transient. Retry silently. |
| 404 (provisioning) | Genuine error. During provisioning or update, a 404 from a downstream dependency indicates a data inconsistency: the API validates all references at creation time, so a dependency that was present at create time should not disappear mid-provisioning except via a deletion event (which would trigger this resource's own deprovisioning). Surface as `ConditionReasonErrored`. |
| 404 (deprovisioning) | Silently accepted. During teardown, a 404 from a downstream service means the resource is already gone — the cleanup goal is already achieved. Do not surface as an error. |
| 403 | Genuine error. A permission failure will not resolve without intervention. |
| 401 | Transient for a bounded number of retries. If the condition persists, surface as `ConditionReasonErrored`. No core library primitive for retry counting exists; controllers that implement this must track retries themselves (e.g. via an annotation or condition message). |
| 409 | Transient. A write conflict resolves on retry with re-read. |

Controllers must not call downstream services to check whether a dependency is provisioned. Dependency readiness is inferred from local status conditions populated by status propagation upward ([section 5.8](#58-status-propagation)). Actively polling a downstream service for readiness reintroduces the tight coupling the event-driven model exists to eliminate.

### 8.8 Deletion Deadlock Detection

A resource that holds a deletion timestamp for an extended period with inbound references still present is in a deadlock: the holder is not releasing its reference, and the resource cannot complete deletion until it does.

**Current state:** There is no platform-level deadlock detection built into the controller framework. Detection is currently done ad-hoc (UI dashboards checking for resources in DELETING state beyond a threshold; kube-state-metrics). The intended direction is to internalize this as a generic platform mechanism that emits a structured event to a configurable sink when a deadlock threshold is crossed, eliminating the need for per-deployment workarounds.

**Operator guidance (interim):** A resource holding a deletion timestamp for longer than an hour with references present is anomalous. Forcibly clearing a reference to resolve a deadlock must only be done by an operator who has confirmed the holding controller will not resume and attempt to use the resource.

### 8.9 Status Projection and Monitoring

Controllers are event-driven: resource status is updated in response to watch events and reconcile passes. This model is correct for lifecycle state that the platform itself controls — provisioning, deletion, reference management.

It is not sufficient for state that lives in the cloud provider and changes independently of platform operations. Power state, health, and other runtime characteristics of cloud resources can only be known by querying the provider directly. Waiting for a lifecycle event to trigger a reconcile would mean that status reflects the last time the controller ran, not the current reality.

Services that manage cloud resources therefore run a monitor process alongside the controller. The monitor polls provider APIs on a periodic schedule and projects the observed state back into resource status. The controller sets the initial observed state — including initial power state — as part of provisioning; the monitor is responsible for keeping that observed state current on subsequent polls. If the provider is unreachable during a poll cycle, the monitor aborts without modifying status; the last observed state is preserved until the provider becomes reachable again.

A new service that manages cloud resources must account for both. Relying solely on controller reconciles to reflect provider reality is insufficient.

**Condition ownership convention.** The controller and the monitor both write to `status.conditions`. By convention the controller owns lifecycle conditions (Provisioning, Provisioned, Deprovisioning, Deprovisioned, Errored) and the monitor owns observed-state conditions (power state, health). Implementations must respect this partitioning to avoid the last-writer-wins race overwriting a terminal lifecycle condition with a stale observed-state value. This partitioning is not enforced by the framework; it is a required implementation discipline.

Projected status is therefore observational by construction: it reflects the last poll, not the present. It is derived state ([§3.1](#31-authority-and-derived-state)). It must never be the authority for a decision about the controller's *own* backing provider state — above all a destructive one: where a decision turns on whether a provider resource this controller owns exists, query the provider, and do not infer its existence or non-existence from projected status (see [§8.10](#810-authoritative-state-for-deprovisioning)). This is distinct from a *dependency's* readiness, which is by design inferred from propagated status and must not be rechecked by polling the owning service ([§8.7](#87-downstream-error-handling)): there, propagated status is the channel through which the owner's source of truth is surfaced ([§3.1](#31-authority-and-derived-state)), not a mirror of state this controller itself owns.

### 8.10 Authoritative State for Deprovisioning

Deprovisioning acts on real, often irreversible state. Every cleanup step must be:

- **Unconditional at the call site.** The decision to *attempt* a cleanup step is not gated on a readiness condition or on best-effort recorded status. Once a deletion timestamp is set the deprovision path runs ([§8.5](#85-finalizer-lifecycle)); it delegates each cleanup step unconditionally.
- **Idempotent and partial-state tolerant.** The step re-discovers what actually exists from the source of truth ([§3.1](#31-authority-and-derived-state)), treats an already-absent resource as success (the 404-on-deprovision rule, [§8.7](#87-downstream-error-handling)), and treats a dependency that was never realised as a no-op rather than an error.
- **Independently gated.** Where teardown has multiple steps — provider cleanup, reference release, allocation/quota release — each is gated only on the concrete state *it* needs. A single broad precondition covering all of them is a defect: it couples independent steps and lets one unmet condition skip the others.

> **Rule:** A cleanup step must not be skipped because recorded status does not mention the resource. Recorded status is derived state; absence of a record is not evidence of absence of the resource ([§3.1](#31-authority-and-derived-state)). Re-discover from the source of truth and act idempotently.

This does not conflict with [§8.7](#87-downstream-error-handling). That rule forbids polling a *downstream service* to learn a *dependency's provisioning readiness* — coupling that status propagation ([§5.8](#58-status-propagation)) replaces. The present rule concerns a controller acting on *its own* owned backing state during teardown, for which the owning system — the cloud provider, the allocator — is the source of truth and must be consulted rather than trusted to match a local mirror. Readiness of someone else's resource is inferred from propagated status; existence of your own backing resource is rediscovered authoritatively.

> **Rationale:** The failure mode is silent and unbounded. A cleanup gated on recorded status leaks every time the status write was lost (crash, conflict, status reset on a re-reconcile) — and because the controller's finalizer is removed when deprovision returns success, the owning resource is garbage-collected while the orphaned backing resource or allocation persists with nothing left to trigger its cleanup. The leak is then only discoverable by reconciling the provider against the allocator out of band. See [Appendix A.7](#a7-best-effort-status-used-as-deprovision-authority).

---

## 9. Observability

### 9.1 Structured Logging

All platform services must use Uber zap for structured JSON logging. Log output is to stdout.

- Controller processes must initialise the controller-runtime logger with a zap backend (`sigs.k8s.io/controller-runtime/pkg/log/zap`) at startup, and retrieve it via `log.FromContext()`.
- All context must be expressed as discrete structured zap fields. Free-form message strings must contain only a human-readable summary — all queryable detail belongs in fields. A log entry whose diagnostic value depends on parsing its message string is a defect.
- **Controllers log verbosely.** Every reconcile pass, every state transition, every reference operation, every requeue decision must be logged at an appropriate level with the resource type and resource ID as structured fields on every entry.
- **API handlers log on non-2xx responses only.** A non-2xx response is always logged with the HTTP status, error code, and trace ID as structured fields.
- All log entries must carry the component name and version as structured fields, matching the audit log schema.

### 9.2 Audit Logging

Audit logging uses the existing Uber zap structured logging infrastructure. An audit log entry is identified by `msg: audit` in the log stream and is always emitted at info level.

Audit logging is mandatory for all authenticated, scoped, state-mutating API operations.

- GET, OPTIONS, and HEAD requests must not be audit logged.
- Unauthenticated requests must not be audit logged.
- Requests without organisational scope must not be audit logged.

Audit log entries are emitted automatically by `audit.Middleware` (see [section 7.2.2](#722-server-middleware-stack)). Service authors do not write audit log entries manually. The middleware wraps the full post-routing chain and emits the entry after the handler returns, capturing the outcome. Any handler that bypasses the middleware stack will silently produce no audit trail — this is a defect.

Every audit log entry must carry the following structured fields:

| Field | Value |
|---|---|
| `level` | `info` |
| `ts` | RFC3339 timestamp |
| `msg` | `audit` |
| `component` | Service name and version |
| `actor` | Principal subject |
| `operation` | HTTP verb |
| `scope` | `organisationID` and `projectID` resolved from the request context (not parsed from the URL path — in v2 they may not be present in the path) |
| `resource` | Resource type and ID |
| `result` | HTTP status code |

OpenTelemetry trace fields are injected by middleware and will be present on audit log entries, enabling correlation with traces across services.

### 9.3 Distributed Tracing

All platform services must instrument with OpenTelemetry and export spans via OTLP. A span must be created for every inbound API request and every controller reconcile pass. Cross-service calls must propagate the trace context.

- The trace ID must be included in all non-2xx API error responses as the `trace_id` field, regardless of sampling decisions.
- Head-based sampling must be implemented at the public API boundary. The sampling decision is made once per trace at entry and propagated to all downstream spans via the trace context.
- Sampling must never suppress the trace ID from error responses.

### 9.4 Metrics

All platform services must expose Prometheus metrics.

**Controllers** must emit at minimum:
- Reconcile duration (histogram)
- Reconcile outcome (counter by outcome: success, transient, error)
- Work queue depth (gauge)
- Reference operation count (counter by operation: add, remove, and outcome) — not yet standardised in the core library; recommended for services with high reference contention

**API handlers** must emit at minimum:
- Request duration (histogram by endpoint and status class)
- Request count (counter by endpoint and status class)

Metric naming must follow Prometheus conventions: `snake_case`, service-prefixed, with unit suffixes where applicable (e.g. `_seconds`, `_total`). Shared metric definitions belong in the core library.

---

## 10. Security

The platform enforces three tiers of access control, each with a distinct authentication mechanism:

| Tier | Principals | Authentication |
|---|---|---|
| Public | End users, service accounts, and tooling | OIDC tokens. Access only via service REST APIs. |
| System | Platform services (system accounts) | mTLS exclusively. Certificate CN maps to RBAC role. |
| Platform | Cloud provider APIs | Accessed only by the Region service via stored credentials. |

RBAC as defined in [section 4.3](#43-rbac) is the enforcement mechanism for Public and System tier access. No principal in any tier may access resources outside their permitted scope as defined by the ACL.

### 10.1 Platform Security Invariants

The following invariants apply unconditionally to all platform components. Any implementation that violates them is a defect, not a design trade-off.

- **No privilege escalation** — no operation may grant a principal more access than they already hold. Impersonation can only narrow a user's effective permissions (via ACL intersection with the service's own ACL); it can never expand them. See [section 4.6.1](#461-acl-intersection-under-impersonation).
- **Principle of least authority** — each actor (service or user) operates with only the permissions required for the current operation. Services must not accumulate permissions beyond their functional scope, and must not forward a user's full ACL when acting as a proxy.
- **Scope confinement** — access granted at a narrower scope (project, organisation) does not implicitly confer access at a broader scope. The scope hierarchy is strictly one-directional: broader grants cover narrower scopes, never the reverse. See [section 4.3](#43-rbac).
- **Single enforcement point** — all access decisions are made against the ACL returned by the identity service. There is no local policy evaluation in individual services. Duplicating or caching access logic outside the ACL endpoint is a defect.
- **Immutable attribution** — the principal recorded on a resource at creation time cannot be changed. Re-attribution is not permitted. See [section 4.4](#44-principals-and-proxies).
- **Header stripping at ingress** — the platform ingress strips `X-Principal` and `X-Impersonate` headers from all inbound external requests before they reach any service. End users cannot inject or spoof principal propagation headers. The mTLS-based principal propagation model is only secure if the ingress enforces this boundary unconditionally.
- **Input-path authorization (taint checking)** — A resource ID supplied by a caller in any position (URL path parameter, query parameter, request body field) is an untrusted input. The caller knowing a valid ID does not imply they are authorized to reference it. Every endpoint that accepts a caller-supplied resource ID must verify authorization for that specific resource before any business logic executes. Applying an ACL filter to a list response is not a substitute for this check: it covers only the output path. An attacker who learns a restricted resource's ID by any means — prior access, traffic observation, or enumeration — must be denied on the input path with the same outcome as if the resource did not exist. Any handler that performs an ACL check on its list response but not on its write or read-by-ID endpoints is a defect.
- **Cache scope isolation** — A response produced under a caller-scoped authorization context (RBAC filtering, impersonation, organisation-scoped visibility) must never be served from a cache whose key is coarser than the full authorization scope. If the authorization scope cannot be fully expressed as a cache key, the cache must not exist at that layer. Caching of caller-scoped responses must be owned exclusively by the service that owns the authorization context. A downstream consumer that receives impersonated or filtered responses must not cache them — the cache key would be under-specified and a principal with broader access to the same key would serve restricted data to a principal with narrower access.

---

## 11. Core Library

Before implementing any pattern in a platform service, check the core library. Reimplementing existing patterns is a defect.

The core library is not a deployed service. It is a shared Go library that provides the canonical implementations of all platform design patterns. If a pattern is described in this specification, its reference implementation is in the core library. The package map below is the index — read the package README before writing any cross-cutting logic.

| Package | Provides |
|---|---|
| `pkg/apis/unikorn/v1alpha1` | Managed resource interfaces (`ManagableResourceInterface`, `ReconcilePauser`, `StatusConditionReader`, `StatusConditionWriter`). Shared value types: `SemanticVersion`, `IPv4Address`, `Tag`. The canonical condition vocabulary. All resources participating in the controller/provisioner lifecycle must implement these interfaces. |
| `pkg/client` | Kubernetes client construction. mTLS HTTP client setup from cert-manager secrets. Do not construct clients ad hoc. |
| `pkg/constants` | Platform-wide metadata label keys and annotation keys. The `Finalizer` constant. `DefaultYieldTimeout`. `LabelPriorities` — the canonical ordering for label-tuple identity paths. Any new label or annotation key must be added here. |
| `pkg/errors` | Sentinel errors: `ErrConsistency`, `ErrResourceNotFound`, `ErrConflict`, `ErrTypeConversion`. Wrap with `%w`; branch with `errors.Is`. |
| `pkg/manager` | Controller-runtime integration. The generic `Reconciler`. Resource reference helpers. `ResourceReady()` — returns `ErrYield` when a dependency is not yet provisioned; use this rather than polling a downstream service. (`ErrYield` is defined in `pkg/provisioners` but emitted from here — both packages must be understood together.) |
| `pkg/messaging` | Event bus publisher and consumer. Use for all cross-service deletion and status propagation. Do not subscribe to another service's storage directly. |
| `pkg/options` | Process bootstrap: namespace, logging (zap + controller-runtime), OpenTelemetry (tracer, meter, OTLP export). All services use this as their startup base. |
| `pkg/provisioners` | The `Provisioner` interface and `ErrYield`. Combinators: `serial` (ordered, stop on first yield), `concurrent` (parallel, independent children), `conditional` (provision or actively deprovision based on a predicate). Use combinators to compose complex provisioning logic rather than writing ad hoc orchestration. |
| `pkg/server/conversion` | `NewObjectMetadata` (create path, random UUID v4) and `NewDeterministicObjectMetadata` (create path, UUID v5 from caller-supplied namespace UUID and invariant string) for constructing platform-standard resource metadata. `UpdateObjectMetadata` for mutation on update. Status mapping: deletion takes precedence over provisioning state. Tag conversion between Kubernetes and OpenAPI representations. |
| `pkg/server/errors` | Canonical error constructors: `HTTPNotFound`, `HTTPConflict`, `AccessDenied`, `OAuth2InvalidRequest`. `HandleError` normalises arbitrary failures to the platform error contract. `PropagateError` adapts generated OpenAPI client error responses. Never construct error responses manually. |
| `pkg/server/middleware` | Pre-routing middleware components: OpenTelemetry, logging, route resolver, CORS, timeout. These are the components assembled in §7.2.2. |
| `identity/pkg/middleware/audit` | `audit.Middleware` — post-routing middleware that emits structured audit log entries. Lives in the Identity service, not core, because audit logging depends on identity-specific principal and ACL context. Import from `github.com/unikorn-cloud/identity/pkg/middleware/audit`. |
| `pkg/server/saga` | Synchronous in-process saga coordinator. Best-effort rollback on failure. Use for any handler that produces side effects across multiple systems. |
| `pkg/util` | `GenerateResourceID` — Kubernetes-safe random UUID v4. `GenerateDeterministicResourceID(idNamespace, invariant)` — Kubernetes-safe UUID v5 derived from a caller-supplied namespace UUID and invariant string; each resource type must use its own fixed namespace UUID constant to prevent cross-type collisions. `ServiceDescriptor` — service name/version/revision for identity in logs and telemetry. |

---

## 12. Glossary

| Term | Definition |
|---|---|
| ACL | Computed set of endpoint scopes and CRUD permissions for a principal |
| ClusterManager | Isolated Cluster API instance; multi-tenant cluster lifecycle management |
| CN | X.509 Common Name — service identity for mTLS; maps to an RBAC role |
| ComputeInstance | User-visible resource owned by the Compute service; realised via a hidden Region Server |
| CRD | Custom Resource Definition — Kubernetes extension for all platform resource types |
| Data Boundary | Each service exclusively owns its data; others access only via versioned API |
| DELETING | Deletion phase: `DeletionTimestamp` set, blocks while reference finalizers exist. API-visible condition: `ConditionReasonDeprovisioning`. |
| DRAINING | Deletion phase: all reference finalizers cleared, reconciler cleaning owned children. API-visible condition: `ConditionReasonDeprovisioning`. |
| FINALIZING | Deletion phase: children gone, reconciler releasing outbound references and removing its own finalizer. API-visible condition: `ConditionReasonDeprovisioning`. |
| DELETED | Deletion state: all references released, resource garbage collected |
| Group | Organisation-scoped collection of users and service accounts; the primary route to RBAC role assignment |
| OAuth2Provider | Identity service resource representing an organisation's upstream OIDC provider configuration; the federation target for user authentication |
| OAuth2Client | Identity service resource representing a client application permitted to initiate the OAuth2 authorisation code flow |
| Pause | Operator annotation that halts all controller reconciliation on a resource. The resource remains in its current condition until unpaused. |
| Tag | User-defined key-value metadata on a resource (`spec.tags`). User-defined values are opaque to the platform; platform-reserved keys in service namespaces (e.g. `compute.unikorn-cloud.org:instance-id`) are used for internal resource coordination. Filtered post-list in-process. Does not affect RBAC, scoping, or storage queries. |
| Identity (Region) | Cloud project/user/credentials provisioned by the Region service for a consuming service |
| mTLS | Mutual TLS — both sides present certificates; all service-to-service auth |
| OIDC | OpenID Connect — end-user authentication protocol |
| Containment Edge | Edge carrying scope propagation and (opt-in) forward deletion propagation; resource cannot exist outside the scope of its container |
| Principal | Originating end-user or service account responsible for a resource; determines quota and billing |
| Consumption Edge | Edge carrying reverse deletion propagation (blocking) and co-location; resource uses another it does not contain |
| Dependency Edge | Edge carrying reverse deletion propagation (triggering) and co-location; deletion of target triggers deletion of source |
| `ErrYield` | Sentinel error signalling expected transient blocking; triggers fixed-period requeue, not exponential backoff |
| `ConditionReasonCancelled` | Condition reason set when a reconcile is interrupted by context cancellation (controller shutdown). The resource retains this condition until the next reconcile pass, at which point it is overwritten by the normal outcome. |
| Project | Organisational workspace; root scope for all user resources |
| Protected Role | RBAC role hidden from public API; grantable only via Helm values |
| Proxy | A service acting on behalf of a principal to provision resources |
| Reference | Named dependency claim on a resource implemented as a Kubernetes finalizer on the target; blocks deletion until removed |
| Region | A registered cloud provider instance |
| ServiceAccount | Organisation-bound non-human identity managed by the Identity service; authenticated via bearer token; authority derived from group membership |
| SigningKey | Identity service resource holding the active and previous JWT signing key pair; governs token issuance and rolling verification |
| Status Propagation | Edge property: state change on the target enqueues the source for reconciliation; source re-derives its own status from aggregate state of connected targets |
| System Account | A platform service identity, authenticated exclusively via mTLS; certificate CN maps to an RBAC role |
| Core Library | Shared Go library of canonical platform patterns; the reference implementation of all platform design patterns |
| `GenerateResourceReference` | Core library function producing a deterministic reference string: `{resource}.{group}/{uuid}`. The canonical key for allocations and resource references. |
| UUID v4 | Randomly generated universally unique identifier. Standard format for all resource names on the platform. |
| UUID v5 | Deterministically generated UUID (RFC 4122). Used when a resource has a natural uniqueness key — either as the resource's own name (`GenerateDeterministicResourceID`) or as the name of a uniqueness-index CRD. See Appendix A.1 for the two approaches and when to choose each. |
| `unikorn-client-issuer` | Root CA for all inter-service mTLS |
| Delegated Principal | A user principal propagated by a proxy service across an internal API boundary. The transport is mTLS; the actor for RBAC, quota, billing, and audit is the user, not the proxy. See [section 4.5.1](#451-api-authentication-classifications). |
| `x-hidden: true` | OpenAPI operation extension that suppresses an endpoint from public documentation (Mintlify). Documentation marker only — no middleware enforcement effect. |

---

## Appendix A: Known Issues

### A.1 TOCTOU Race — Resource Name Uniqueness

Resources with a natural uniqueness key (e.g. hostname on a virtual network) must not use list-then-check for duplicate detection — the window between the read and the write is a race. Two approaches exist; the choice depends on whether the resource's UUID needs to be stable or fresh across recreations.

**Approach A — Deterministic resource ID** (`NewDeterministicObjectMetadata`)

The resource itself is given a UUID v5 name derived from its natural key (e.g. `(networkID, hostname)`). Two concurrent creates with the same natural key produce the same UUID and collide natively at the storage layer — one receives a `409`, no application-level race detection required. No separate index resource is needed.

Trade-off: if the resource is deleted and recreated with the same natural key, it gets the same UUID. Billing and audit records from the previous lifetime share the identifier with the new resource.

Each resource type must define its own fixed namespace UUID constant to prevent cross-type collisions. The invariant must be composed of stable, immutable fields.

**Approach B — UUID v5 index resource** (original fix)

Introduce a `HostnameIndex` CRD whose sole purpose is holding a uniqueness slot. Its name is UUID v5 of the natural key; the main resource retains a random UUID v4. The index entry is created first (in a saga); a `409` on index creation returns `409` to the caller. The resource is set as Kubernetes owner of its index entry so GC handles cleanup.

Trade-off: more moving parts (extra CRD, owner reference, saga step), but the main resource gets a fresh UUID v4 on each creation — unambiguous billing and audit records across lifetimes.

**Choosing between approaches:** prefer Approach A for new work — simpler, fewer CRDs, no saga dependency. Use Approach B only when UUID stability across recreations would cause a correctness problem (e.g. billing systems that key records on resource UUID and cannot tolerate reuse).

`isServerNameInUse()`, `isInstanceNameInUse()`, and any equivalent list-then-check functions are redundant under either approach and must be deleted.

### A.2 Pagination — Known Deficiency

**Risk:**
- Unbounded list responses consume memory proportional to the result set size. At sufficient scale this will cause out-of-memory conditions in the handler process.
- Large result sets increase response latency.
- Kubernetes label selector queries that return very large result sets place load on the etcd backend, degrading all concurrent operations.

Pagination has not yet been implemented. This is a known scaling limitation.

### A.3 Quota Allocation — Async Boundary Violation

**Status: under active review — do not replicate this pattern.**

**Current non-compliant pattern:**
- Handler saga step 1: `createAllocation` (committed). Compensating transaction: `deleteAllocation`.
- Handler saga step 2: create resource CRD. No compensating transaction.
- Controller Deprovision: releases allocation.

**Defect:** the handler commits the allocation directly rather than creating a soft reservation. This violates the two-phase model in [section 7.6](#76-quota-and-strongly-consistent-allocations) — committed allocation must be owned by the controller, not the handler.

### A.4 Update Saga Revert — Broken Compensation

**Status: under active review — do not replicate this pattern.**

**Correct pattern:**
- Forward action: update allocation to values derived from the **new** resource spec.
- Compensating transaction: update allocation to values derived from the **original** resource spec (pass the current resource, not the updated one, to the revert call).

**Defect:** current implementations pass the updated resource to the compensating transaction, reverting to the new values rather than the original values. The compensation does not restore the prior state.

### A.5 Multi-Step Create With No Saga — Orphaned Cross-Service Resources

**Status: under active review — do not replicate this pattern.**

**Current non-compliant pattern:**
- Step 1: `createAllocation` — commits quota in identity service. No compensation registered.
- Step 2: `createIdentity` — creates cloud identity in region service. No compensation registered.
- Step 3: `applyCloudSpecificConfiguration` — conditionally creates additional region resources. No compensation registered.
- Step 4: `c.client.Create` — creates the local CRD. Terminal write. No rollback of steps 1–3 on failure.

**Required pattern:** use a saga. Register a compensation action for each side-effectful step before executing it. Step 1 compensation: delete allocation. Step 2 compensation: delete identity. The saga commits only after the terminal CRD write succeeds. On any failure, the saga executes registered compensations in reverse order.

### A.6 Allocation Ownership Inversion

**Status: under active review — do not replicate this pattern.**

**Symptoms:**
- `handler.Delete` calls `identityclient.NewAllocations.Delete` before `c.client.Delete`.
- `provisioner.Deprovision` does not call `identityclient.NewAllocations.Delete`, or contains a TODO noting the omission.

**Required pattern:** `handler.Delete` must not release the allocation. It deletes only the local resource (setting the deletion timestamp). The controller's deprovision path is solely responsible for releasing the allocation before removing its finalizer.

### A.7 Best-Effort Status Used as Deprovision Authority

**Status: resolved in Region; recorded so the pattern is not reintroduced.**

**Symptoms:**
- A deprovision step is gated on a predicate that reads projected status — e.g. `if statusRecordsProviderResourceIDs(resource) { provider.Delete(...) }` — or on parent readiness, so the destructive call is skipped when status is empty.
- A single broad predicate gates several unrelated teardown steps (provider cleanup, reference release, allocation release) together.
- Symptom in the field: an allocation or backing cloud resource survives after the owning resource is gone, discoverable only by reconciling the provider/allocator out of band.

**Defect:** projected status is derived state ([§3.1](#31-authority-and-derived-state)). It is written best-effort and after the side effect it records, so a lost write (crash, resource-version conflict, or a reconcile that resets status before re-deriving it) makes the predicate read "nothing was created" when something was. The cleanup is skipped, the controller's finalizer is removed on apparent success, and the owning resource is garbage-collected while the real resource leaks. Concrete instance: a provider-network VLAN was allocated and the backing network create then failed before status was persisted; on the cascade delete the status-gated provider cleanup was skipped and the VLAN allocation was orphaned.

**Required pattern:** delegate each deprovision step unconditionally and make it idempotent ([§8.10](#810-authoritative-state-for-deprovisioning)). The provider rediscovers its resources from the source of truth (by stable name/tag) and frees allocations keyed by stable identity; an already-absent resource or a never-realised dependency is a no-op, not a skip and not an error. Gate each teardown step only on the concrete state it needs, never on a shared best-effort precondition.

---

## Appendix B: Reference Checklists

### B.1 Checklist: Building a New Service

Use this checklist when introducing a new service to the platform. Each item maps to a section of this specification. An item is complete only when the implementation is in code, not merely planned.

**API Design**
- [ ] OpenAPI spec written first, before any Go code ([§7.11](#711-openapi-first-development))
- [ ] Internal endpoints annotated with `x-hidden: true` (documentation visibility only); access restricted by RBAC policy; handler resolves principal per §4.5.1 and §4.6 ([§7.2.1](#721-handler-authentication-classification))
- [ ] All possible `4xx`/`5xx` responses declared in the OpenAPI spec ([§7.9](#79-error-handling-and-propagation))
- [ ] API follows the v2 flat routing model; no organisation/project in URL path ([§7.10](#710-v2-api-design-model))
- [ ] Server stubs, client types, and validator generated from the OpenAPI spec

**Handlers**
- [ ] Every handler satisfies the middleware prerequisites, general invariant, and per-operation rules of [§7.2](#72-handler-layer-responsibilities)
- [ ] Labels and annotations set exclusively via `conversion.NewObjectMetadata` / `conversion.UpdateObjectMetadata` ([§6.2](#62-labels-and-annotations))
- [ ] All three label categories populated: scope, ancestry, attribution ([§6.2](#62-labels-and-annotations))
- [ ] Principal propagated into context before any downstream call ([§4.5](#45-principal-propagation))
- [ ] v2 services: organisation and project completed before any downstream call — root resource from request body, child resource from parent resource labels ([§4.4](#44-principals-and-proxies), [§7.10](#710-v2-api-design-model))
- [ ] Every user-supplied resource ID (URL param, query param, body field) has an authorization check before business logic executes ([§7.2](#72-handler-layer-responsibilities), [§10.1](#101-platform-security-invariants))
- [ ] ACL-restricted or scoped-visibility resources return `404` for both non-existence and access denial — never `403` ([§7.9](#79-error-handling-and-propagation))
- [ ] No caching of caller-scoped (impersonated, RBAC-filtered) responses in downstream consumers ([§10.1](#101-platform-security-invariants))
- [ ] Multi-step creates use a saga with compensating transactions for each side-effectful step ([§7.5](#75-multi-step-operations-sagas))
- [ ] Per-network uniqueness enforced via deterministic resource ID or UUID v5 index resource — not list-then-check ([Appendix A.1](#a1-toctou-race--resource-name-uniqueness))
- [ ] Error responses constructed via core library helpers only; never manually constructed ([§7.9](#79-error-handling-and-propagation))

**RBAC and Security**
- [ ] System account CN-to-role mapping configured in Identity service Helm values ([§4.3](#43-rbac))
- [ ] System account role declares only the minimum required `(resource, operation)` pairs ([§4.3](#43-rbac))
- [ ] Ingress configured to strip `X-Principal` and `X-Impersonate` headers from external requests ([§10.1](#101-platform-security-invariants))

**Quota** *(if the service introduces user-accountable resources)*
- [ ] New quota kind declared in Identity service quota schema ([§7.6](#76-quota-and-strongly-consistent-allocations))
- [ ] Default quota limit configured via Helm values at deployment time ([§7.6](#76-quota-and-strongly-consistent-allocations))
- [ ] Handler creates soft reservation; controller promotes and releases ([§7.6](#76-quota-and-strongly-consistent-allocations))

**Resource Graph**
- [ ] All edge types declared: for each relationship, the primitive properties are identified ([§5.10](#510-defining-a-new-resource-type))
- [ ] Scope, ancestry, and attribution labels correct for each edge type ([§6.2](#62-labels-and-annotations))
- [ ] Co-location constraint enforced at create time for consumption and dependency edges ([§5.10](#510-defining-a-new-resource-type))
- [ ] Cross-service deletion propagation uses the event bus, not direct storage subscription ([§5.4](#54-event-bus), [§5.9](#59-deletion-propagation-mechanisms))
- [ ] References placed and released by the controller, never the handler ([§5.3](#53-references))

**Controllers**
- [ ] Each controller type is a separate binary, each calling `pkg/manager.Run` with its own factory ([§8](#8-controller-behaviour))
- [ ] Each reconcile pass checks `DeletionTimestamp` first; finalizer added unconditionally before provisioning ([§8.5](#85-finalizer-lifecycle))
- [ ] Every reconcile pass writes `status.conditions` before returning (except: resource already deleted with no finalizers, or resource is paused) ([§8.4](#84-status-conditions))
- [ ] Transient conditions return `ErrYield`; genuine errors returned as errors ([§8.2](#82-transient-conditions-and-silent-retry), [§8.3](#83-genuine-errors))
- [ ] Reconciliation is idempotent; handles duplicate events without side effects ([§8.1](#81-the-work-queue))
- [ ] Each deprovision step delegated unconditionally and is idempotent; none gated on best-effort recorded status; each teardown step gated only on the concrete state it needs ([§8.10](#810-authoritative-state-for-deprovisioning), [Appendix A.7](#a7-best-effort-status-used-as-deprovision-authority))
- [ ] Status propagation upward implemented via explicit watch or event bus subscription ([§5.8](#58-status-propagation), [§8.6](#86-controller-watches))
- [ ] Deletion behaviour reviewed against §8.8 (platform-level deadlock detection not yet available; document any service-specific mitigations)

**Cloud Resource Services** *(if the service manages cloud provider resources)*
- [ ] Monitor process implemented alongside the controller for observed runtime state ([§8.9](#89-status-projection-and-monitoring))

**Middleware Stack**
- [ ] Server initialised with the canonical pre-routing middleware in order: OTel → Logging → Route Resolver → CORS ([§7.2.2](#722-server-middleware-stack))
- [ ] Post-routing middleware registered in order: `audit.Middleware` first, `validator.Middleware` second ([§7.2.2](#722-server-middleware-stack))

**Observability**
- [ ] Uber zap structured logging initialised; controller-runtime logger set to zap backend ([§9.1](#91-structured-logging))
- [ ] Process bootstrapped via `pkg/options` for logging, OpenTelemetry, and namespace configuration ([§11](#11-core-library))
- [ ] OpenTelemetry spans created for all inbound requests and controller reconcile passes ([§9.3](#93-distributed-tracing))
- [ ] Prometheus metrics exposed: reconcile duration, reconcile outcome, work queue depth ([§9.4](#94-metrics))
- [ ] Audit logging emitted automatically by `audit.Middleware`; no manual audit writes in handlers ([§9.2](#92-audit-logging))

**Data Boundaries**
- [ ] Service accesses no other service's storage directly; all cross-service reads/writes via versioned API ([§3](#3-data-boundaries))
- [ ] Intra-service owner references used for containment; cross-service references use the event bus ([§5.9](#59-deletion-propagation-mechanisms))
