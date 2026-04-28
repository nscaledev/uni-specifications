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
3. [Data Boundaries](#3-data-boundaries)
4. [Identity and Access](#4-identity-and-access)
   - [4.1 Organisational Hierarchy](#41-organisational-hierarchy)
   - [4.2 Actors and Representation](#42-actors-and-representation)
   - [4.3 RBAC](#43-rbac)
   - [4.4 Principals and Proxies](#44-principals-and-proxies)
   - [4.5 Principal Propagation](#45-principal-propagation)
     - [4.5.1 API Authentication Classifications](#451-api-authentication-classifications)
   - [4.6 Principal Propagation Modes and Impersonation](#46-principal-propagation-modes-and-impersonation)
     - [4.6.1 ACL Intersection under Impersonation](#acl-intersection-under-impersonation)
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
8. [Controller Behaviour](#8-controller-behaviour)
   - [8.1 The Work Queue](#81-the-work-queue)
   - [8.2 Transient Conditions and Silent Retry](#82-transient-conditions-and-silent-retry)
   - [8.3 Genuine Errors](#83-genuine-errors)
   - [8.4 Status Conditions](#84-status-conditions)
   - [8.5 Finalizer Lifecycle](#85-finalizer-lifecycle)
   - [8.6 Controller Watches](#86-controller-watches)
   - [8.7 Downstream Error Handling](#87-downstream-error-handling)
   - [8.8 Deletion Deadlock Detection](#88-deletion-deadlock-detection)
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
| Identity | Identity Service | Organisation, Project, User, Group, Role, OAuth2Client |
| Region | Region Service | Region, CloudIdentity, Network, PhysicalNetwork, ServerGroup |
| Compute | Compute Service | Server |
| Kubernetes | Kubernetes Service | ClusterManager, Cluster |
| Core | Core Library | (shared library — no resources) |

### 2.2 Service Dependency Order

```
Identity Service  ←  Region Service  ←  Compute Service
                                     ←  Kubernetes Service
```

---

## 3. Data Boundaries

Each service exclusively owns its data. No other service may read or write that data except via the owning service's versioned REST API.

- Storage format is an implementation detail of the owning service.
- Storage must support tombstones, reference-based deletion ordering, and optimistic locking.

> **Rationale:** If two services share a database or read each other's storage directly, they become coupled at the data layer — schema changes in one break the other, deployment order matters, and the boundary between their responsibilities blurs. Enforcing data boundaries through versioned APIs is the only way to keep services independently deployable and independently understandable.

> **Rule:** Never access another service's CRDs or storage directly. Any code that does so is a defect.

---

## 4. Identity and Access

### 4.1 Organisational Hierarchy

The top-level organisational unit is the Organisation. Organisations contain one or more Projects. All user-managed resources are owned by a Project.

The hierarchy is: **Organisation → Project → Resources**. This is not merely a naming convention — it is the structural spine of the access model. Permissions granted at a higher level flow down to all nodes below.

An Organisation is the boundary of tenancy. Resources in one Organisation are never visible to principals of another Organisation unless explicitly shared.

### 4.2 Actors and Representation

Three distinct classes of actor interact with the platform:

| Actor | Representation |
|---|---|
| End User | A human operator authenticated via OIDC. Represented by a bearer token. Identity is established by token introspection at the public API boundary. |
| Service Account | A platform service acting autonomously. Represented by an X.509 certificate whose Common Name maps to an RBAC role. All communication is exclusively via mTLS — the certificate is the identity. |
| Proxy Service | A platform service acting on behalf of an end user. Carries both its own service identity (mTLS) and the originating user's principal (propagated in context). See [section 4.4](#44-principals-and-proxies). |

### 4.3 RBAC

RBAC roles are composed of endpoint scopes, each with a CRUD permission set. Roles may be scoped at three levels: global, organisation, or project.

The ACL endpoint returns the union of all role scopes for a principal. Every service calls this endpoint to authorise operations — there is no local policy evaluation.

Protected roles (`protected: true`) are not visible via the public API and may only be granted via Helm values at deployment time. These roles gate privileged platform operations.

Every public API endpoint requires authentication. The only unauthenticated endpoints are OIDC discovery and login flows.

### 4.4 Principals and Proxies

A **principal** is the originating end-user responsible for a resource. The principal determines quota attribution and billing. Once set on a resource it is immutable.

A **proxy** is a service provisioning resources on behalf of a principal. The proxy carries its own service identity for transport authentication, but the principal it is acting for is propagated separately in the request context.

At the public API boundary, the principal is derived from token introspection (the actor claim) combined with the organisation and project from the HTTP path. This derivation happens once, at the boundary, and the result is threaded through all downstream processing.

A proxy may provision resources in its own tenancy or in the principal's tenancy, depending on whether direct end-user access to those resources is appropriate.

### 4.5 Principal Propagation

- Every outbound service-to-service call must propagate the principal from the current request context.
- Every resource created must persist the principal as an immutable field.
- Every controller calling another service must propagate the principal from the resource under reconciliation — not from the service's own identity.
- Quota allocations must be made against the principal, never the proxy service.

#### 4.5.1 API Authentication Classifications

Every API endpoint falls into exactly one of three authentication classifications:

**Public** — OAuth2 bearer token required. The principal is derived at the boundary from token introspection. The endpoint is reachable by authenticated tenants. No mTLS client certificate is required from the caller. RBAC is enforced against the derived user principal. OpenAPI annotation: no extension required (this is the default).

**Internal (service principal)** — mTLS client certificate required, no bearer token accepted. The calling service certificate CN is the actor. No user principal is present or expected. RBAC is enforced against the service identity. Used for purely platform-initiated operations with no user context. OpenAPI annotation: `x-internal: true`.

**Internal (delegated principal)** — mTLS client certificate required, no bearer token accepted. A user principal must be propagated in the request context by the calling service. The mTLS CN identifies the authorised relay — the service trusted to carry the principal — but is not the actor for RBAC or audit. RBAC is enforced against the propagated user principal. Quota, billing, and audit are attributed to the user. OpenAPI annotation: `x-internal: true`, `x-principal: delegated`.

### 4.6 Principal Propagation Modes and Impersonation

A user principal (`X-Principal`) may be propagated on any authenticated service-to-service call. Its presence alone does not determine how the receiving service uses it. Two explicit modes govern this, signalled by the presence or absence of `X-Impersonate`:

**Attribution-only propagation** — `X-Principal` is present; `X-Impersonate` is absent or false. The receiving service records the principal for billing, quota, and audit attribution only. It does not resolve RBAC against the propagated principal. Access decisions are made against the calling service's own mTLS certificate identity. This is the required mode for controllers provisioning resources on behalf of a user, where authorisation was established at the public API boundary and does not need to be re-evaluated on every downstream call.

**Impersonation** — `X-Principal` is present and `X-Impersonate: true` is also set. The receiving service treats the propagated principal as the authoritative actor for RBAC, quota, billing, and audit on that request. The effective ACL is the intersection of the user's ACL and the impersonating service's own system account ACL (see [ACL Intersection under Impersonation](#acl-intersection-under-impersonation) below). This mode is only appropriate for proxy services presenting a user's identity downstream so that the full access control chain is preserved end-to-end.

**Rules:**

- `X-Principal` may be included on any authenticated service-to-service call for billing, quota, and audit attribution. Its presence alone does not trigger RBAC impersonation.
- RBAC impersonation is activated only when `X-Impersonate: true` accompanies `X-Principal`.
- In attribution-only mode, the receiving service resolves RBAC against the calling service's mTLS certificate identity.
- Impersonation (`X-Impersonate: true`) is only appropriate for proxy services. It is not a general-purpose capability.
- A service must never impersonate another service's identity. The mTLS certificate CN is fixed. Only the user principal context may travel in `X-Principal`.
- Any service holding a valid platform CA-signed mTLS certificate is trusted to signal impersonation. No additional allowlist of permitted impersonators is maintained. The mTLS trust boundary is the sole control.

#### ACL Intersection under Impersonation

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
| Containment | Scope Propagation + Forward Deletion Propagation (opt-in) + Status Propagation Upward (where applicable) + Co-location (implicit) | Organisation → Project; Cluster → Server |
| Consumption | Reverse Deletion Propagation (blocking) + Status Propagation Upward (where applicable) + Co-location Required | Server → Network |
| Dependency | Reverse Deletion Propagation (triggering) + Status Propagation Upward (where applicable) + Co-location Required | Cluster → Network |

A new resource type that does not fit these three patterns is expressed by declaring which primitive properties its edges carry — no new named types are required.

### 5.3 References

All edge relationships are expressed as named references on the target resource. A reference blocks deletion of the resource it is placed on. A resource in DELETING state blocks until all references are removed. References do not lapse automatically.

References may cross service boundaries and are managed via the reference API. Reference operations are idempotent:

```
PUT    /api/v1/{resource}/references/{name}  →  200 OK
DELETE /api/v1/{resource}/references/{name}  →  200 OK
```

The canonical reference string is derived by `GenerateResourceReference`: `{resource}.{group}/{uuid}`. This is deterministic, stable for the lifetime of the consuming resource, and unique across all resource types and groups. No other format is permitted for cross-resource references.

Adding a reference is idempotent — if already present the operation is a no-op. Removing a reference is idempotent — if absent the operation is a no-op. Both operations use optimistic locking; a write conflict is a transient condition and must be silently retried (see [section 8.2](#82-transient-conditions-and-silent-retry)).

References are placed by the consuming resource's controller and removed by the same controller. References are never placed or removed by the handler layer.

Two valid reference lifecycle patterns exist:

**Permanent hold** — the reference persists for the full lifetime of the consuming resource. Used when the dependency is only expressible at the Kubernetes level. Lifecycle: place reference → use resource → remove reference as part of deprovisioning → remove finalizer. The consuming resource's own finalizer must not be removed until all outbound permanent references have been released.

**Transient provisioning guard** — the reference is placed at the start of a Provision pass and removed once the downstream infrastructure has taken ownership of the live dependency. Used when the consumed resource is referenced by external infrastructure after provisioning completes, and the external system is the authoritative guard against deletion during normal operation. On subsequent reconcile passes, the guard is re-applied and released again around the provisioning call.

### 5.4 Event Bus

The event bus is a push-based messaging system through which services publish resource lifecycle events and subscribe to events from services they depend on. Its current implementation is Kubernetes list/watch, but it is designed for replacement with a scalable enterprise offering (e.g. Confluent) without changing the event semantics.

- On service start or restart, all existing resources on the subscribed topic must be replayed. This allows subscribers to trigger or re-evaluate any deletion that was missed during downtime. All subsequent events are streamed.
- Event delivery is at-least-once. Subscribers must handle duplicate events idempotently. Exactly-once delivery is not guaranteed and must not be assumed.
- Event payloads are minimal: `resourceID` and optional `deletionTimestamp`. No resource state is embedded in the event. Subscribers must call the owning service's versioned API to retrieve current state. This preserves data boundaries — the event is a stimulus, not a data transfer.
- The `deletionTimestamp` field allows subscribers to filter the stream and distinguish deletion events from creation or update events without an additional API call.
- The event bus is the required mechanism for all cross-service deletion and status propagation. Intra-service propagation uses Kubernetes watches directly. The semantics are identical — only the plumbing differs.

### 5.5 Containment

A containment edge expresses that a resource cannot exist outside the scope of its container. The contained resource's existence is logically bounded by the container. Example: an Organisation contains a Project; a Project contains a Server (via transitive closure).

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

### 5.12 Deletion State Machine

| State | Description |
|---|---|
| DELETING | Resource tombstoned. Deletion event published. Blocks while any inbound references exist. |
| DRAINING | All inbound references removed. Cleaning up contained child resources. |
| FINALIZING | Contained children gone. Releasing outbound references held on consumed or depended-on resources. |
| DELETED | All references released. Resource garbage collected. |

### 5.13 Deletion Events

- Deletion events must be handled idempotently. Duplicate events are expected.
- Event payloads carry only `resourceID` and `deletionTimestamp`. No resource state is embedded. Subscribers must call the owning service's API to get current state.

---

## 6. Resource Model

### 6.1 Naming and Metadata

- Resource identifiers are UUID v4 (random), generated at creation time. They are globally unique and immutable for the lifetime of the resource. A resource ID is never reused, reassigned, or modified. UUID v5 (deterministic) is used only for index resources whose sole purpose is uniqueness enforcement — never for resources carrying billing, quota, or audit identity. See [Appendix A.1](#a1-toctou-race--resource-name-uniqueness).
- Human-readable names are stored in `metadata.labels[<platform-label-prefix>/name]` and are mutable. Never hardcode resource names.
- Resource status is expressed exclusively via `status.conditions`, following the `metav1.Condition` schema with strongly-typed reason constants. There is no authoritative phase field — conditions are the sole source of truth for resource state.
- All write operations are asynchronous (202 Accepted). Always read `status.conditions` before acting on resource state.

### 6.2 Labels and Annotations

- Labels and annotations are exclusively owned by the server handler layer (`generate()` / `conversion.NewObjectMetadata`).
- No controller, external process, or agent may add, modify, or remove them. Any externally-written values are silently overwritten on the next write.
- Changes to label or annotation semantics require a revision to this specification before implementation.

---

## 7. API Design

### 7.1 Synchronous vs Asynchronous Operations

The distinction between synchronous and asynchronous operations reflects whether an operation can be fully effected within the request/response cycle, or whether it initiates work that requires a controller to drive to completion.

**Synchronous** operations are ones whose effect is immediate and complete by the time the response is returned. No further processing is required. Examples: reading a resource, applying or releasing a reference.

- Synchronous reads return `200 OK`. Synchronous creates return `201 Created`.

**Asynchronous** operations cannot be completed immediately — they involve provisioning, configuring, or tearing down real infrastructure or dependent resources. Every asynchronous operation must have an accompanying controller responsible for driving the resource to its terminal state.

- Long-running operations must never be performed synchronously.
- Asynchronous operations return `202 Accepted` immediately. The response body contains the resource with its initial transitional conditions set. The caller reads `status.conditions` to track progress.
- If a resource already has a deletion timestamp set, return `202` without re-triggering deletion. If the resource is not found, return `404`.

### 7.2 Handler Layer Responsibilities

Every server handler must perform the following steps in order:

1. Introspect the bearer token against the identity service to establish the principal.
2. Retrieve the ACL for the principal and enforce the required scope and CRUD permission for the operation.
3. Validate the request body against the OpenAPI schema (performed by middleware before the handler is invoked).
4. For **create**: derive all labels and annotations via `conversion.NewObjectMetadata`. Never accept labels or annotations from the request body.
5. For **update**: read the current resource (`GetRaw`), re-derive labels and annotations from the current resource's own label values, patch with optimistic locking (`MergeFromWithOptimisticLock`). Never carry labels or annotations from the request body.
6. For **delete**: verify `DeletionTimestamp` is nil before issuing the delete. If already set, return `202` idempotently.
7. Propagate the principal into the request context before any downstream service call or resource write.

#### 7.2.1 Handler Authentication Classification

Every handler must declare its authentication classification via OpenAPI extensions on the operation or path. The classification must match [section 4.5.1](#451-api-authentication-classifications) exactly.

- **Public handlers** (default, no extension): middleware derives the principal from the OAuth2 bearer token. The handler calls `rbac.AllowProjectScope` or `rbac.AllowOrganizationScope` against it.
- **Internal (service principal) handlers** (`x-internal: true`): middleware verifies the mTLS CN against the allowed caller list. No bearer token is accepted. No user principal is present in context.
- **Internal (delegated principal) handlers** (`x-internal: true`, `x-principal: delegated`): middleware verifies the mTLS CN and asserts that a user principal is present in the request context. The handler calls `rbac.AllowProjectScope` or `rbac.AllowOrganizationScope` against the propagated user principal.

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

> **Compliance:** `uni-region` is compliant with this section. Any new service joining the platform must replicate this stack exactly. Deviations must be approved and documented here.

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

The platform quota system tracks consumption of arbitrary, service-defined resource kinds. Quota limits are set at the organisation level.

Quota allocation uses a two-phase reservation model. The handler makes a soft reservation synchronously. The controller promotes the reservation to a committed allocation and owns the full allocation lifecycle.

- **On resource create**: the handler computes the reservation key via `GenerateResourceReference` and checks whether available quota (limit minus committed allocations minus active reservations) is sufficient. If insufficient, the handler returns `403 Forbidden` immediately. If sufficient, the handler creates a soft reservation keyed on the resource UUID before returning `202`. The reservation reduces the available balance immediately, preventing concurrent requests from racing past the quota limit.
- **The controller** promotes the reservation to a committed allocation on the provisioning path. Promotion is idempotent on the key.
- **The controller** releases the committed allocation on the deprovisioning path, before removing its finalizer. Release is idempotent on the key.
- If the resource is deleted before the controller has promoted the reservation, the controller detects the tombstone, releases the reservation rather than a committed allocation, and removes its finalizer. The quota balance is restored correctly regardless of which phase the lifecycle was in.
- Reservations carry a long TTL sufficient to encompass any realistic provisioning duration. The TTL is a safety net for leaked reservations only — it is not a normal operational path.
- If a reservation has expired before the controller runs, the controller attempts a fresh allocation against the current balance. If quota is now exhausted, the controller sets `ConditionReasonErrored`.
- The quota system must treat all allocation, promotion, and release operations as idempotent on the key.
- The quota service must provide atomic check-and-reserve. The available balance check and the reservation creation must be a single atomic operation.

### 7.7 API Versioning

- Public-facing APIs must be backward compatible within a version. Additive changes — new optional fields, new endpoints, new enum values — are permitted without a version bump. Removing fields, changing field semantics, or altering response structure requires a version bump.
- A deprecated version must remain available for a defined deprecation period before removal. Deprecation must be communicated via API response headers and documented in the changelog.
- Inter-service APIs may be broken deliberately, provided the producing service and all consuming services are updated in the same coordinated release. No deprecation period is required.
- Major version bumps — paradigm shifts rather than incremental changes — should be coordinated across all services simultaneously. The v2 release is the canonical example of a coordinated major bump.

### 7.8 List Filtering

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
- Where revealing that a resource exists would constitute a data boundary violation, return `404` uniformly regardless of whether the actual cause is non-existence or forbidden access. Never use `403` as an existence oracle. (OWASP WSTG-ERRH-01, CWE-208)
- Authentication error responses must not distinguish between an unknown principal and an incorrect credential. Both return `401 access_denied` with a non-specific description. (OWASP WSTG-ERRH-02)
- Error descriptions must never reference resources, identifiers, or state belonging to a scope the requesting principal cannot access.
- When a handler calls a downstream service and receives an error response, it must propagate the typed error faithfully. A `403` from downstream propagates as `403` to the end user.
- Untyped errors are caught by the terminal error handler and written as `500 server_error` with no detail.
- The `401` response includes a `WWW-Authenticate` header per RFC6750 and RFC9728. Services must not construct this header manually — use the canonical `AccessDenied` constructor from the core library.

---

## 8. Controller Behaviour

### 8.1 The Work Queue

The work queue decouples event production from event processing. Events are produced by resource watches and by controllers explicitly requeuing themselves. The queue deduplicates — if a resource is enqueued multiple times before it is processed, it is processed once.

Processing is the reconcile function. It runs to completion and then the item is removed from the queue. There is no streaming, no callback, no blocking wait inside the function — if the controller cannot proceed, it returns and puts itself back on the queue.

- Reconciliation must be idempotent.
- If a resource has been paused, the controller must skip reconciliation entirely and return immediately without requeuing.

### 8.2 Transient Conditions and Silent Retry

A transient condition is any condition that is expected to resolve without intervention and will not produce the same outcome on retry. Controllers must never surface transient conditions to the user as failures. The resource remains in its current in-progress condition, the reconcile returns, and the controller retries after a fixed interval.

Transient conditions include, but are not limited to: a dependency not yet provisioned; references not yet released; a remote resource not yet in the required state; a write conflict (409); a temporary downstream service unavailability (5xx); an event bus stimulus arriving before the resource state is queryable.

- A transient condition must never set the resource's condition to `Errored`.
- The retry interval is fixed and constant. It must not grow.
- `ErrYield` is the implementation mechanism. Returning `ErrYield` from the reconcile function signals the controller runtime to requeue the item after `DefaultYieldTimeout` without treating it as a failure.

### 8.3 Genuine Errors

A genuine error is an unexpected condition that cannot be resolved by requeueing. Examples: a malformed resource that cannot be processed, an unrecoverable failure in a downstream system, an assertion that should never be false.

- Genuine errors are returned as real errors from the reconcile function. The controller-runtime applies exponential backoff.
- A genuine error sets the resource's `Available` condition to `ConditionReasonErrored` and writes the error message to `status.conditions`. This is the correct and only channel for surfacing failures the user needs to act on.
- Genuine errors must be reserved strictly for conditions that are truly unrecoverable without intervention.

### 8.4 Status Conditions

Every pass through the reconcile function, successful or not, must update the resource's `Available` condition before returning. This is unconditional — a reconcile that exits without writing status has failed its contract with the queue.

| Outcome | Condition | Queue behaviour |
|---|---|---|
| Transient (`ErrYield`) | `ConditionReasonProvisioning` or `ConditionReasonDeprovisioning` | Requeued at fixed interval |
| Genuine error | `ConditionReasonErrored` | Requeued with exponential backoff |
| Success | `ConditionReasonProvisioned` or `ConditionReasonDeprovisioned` | Removed from queue |

If the status write itself fails (e.g. due to a resource version conflict), the controller must not return an error. It must requeue with a fixed timeout. A status write failure is a transient queue item — it must not trigger exponential backoff.

### 8.5 Finalizer Lifecycle

- On creation or update: the controller adds its finalizer to the resource before performing any provisioning work.
- On deletion: the resource is tombstoned with a deletion timestamp. The controller detects this, retries silently while references and owned children exist, runs deprovisioning, and only removes the finalizer once deprovisioning is complete.

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
| 404 | Genuine error. A 404 from a downstream service indicates a data inconsistency that will not resolve on retry. Surface as `ConditionReasonErrored`. |
| 403 | Genuine error. A permission failure will not resolve without intervention. |
| 401 | Transient for a bounded number of retries. If the condition persists, surface as `ConditionReasonErrored`. |
| 409 | Transient. A write conflict resolves on retry with re-read. |

Controllers must not call downstream services to check whether a dependency is provisioned. Dependency readiness is inferred from local status conditions populated by status propagation upward ([section 5.8](#58-status-propagation)). Actively polling a downstream service for readiness reintroduces the tight coupling the event-driven model exists to eliminate.

### 8.8 Deletion Deadlock Detection

On every reconcile pass of a resource in DELETING state, the controller checks the age of the deletion timestamp. If the deletion timestamp has been set for longer than the configured deadlock threshold and inbound references are still present, the controller emits a structured error log entry via Uber zap.

The deadlock log entry must include as structured fields: resource type, resource ID, deletion timestamp, elapsed duration, and the full set of blocking reference strings. These fields must be present as discrete zap fields — not embedded in a free-form message string — so they are indexable and queryable in Loki.

The deadlock threshold is **10 minutes**. A resource holding a deletion timestamp for longer than 10 minutes with references still present is anomalous and warrants operator attention.

Forcibly clearing a reference to resolve a deadlock must only be done by an operator who has identified why the reference is held and confirmed that the holding controller will not resume and attempt to use the resource.

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

Every audit log entry must carry the following structured fields:

| Field | Value |
|---|---|
| `level` | `info` |
| `ts` | RFC3339 timestamp |
| `msg` | `audit` |
| `component` | Service name and version |
| `actor` | Principal subject |
| `operation` | HTTP verb |
| `scope` | `organisationID` and `projectID` from path |
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
- Reference operation count (counter by operation: add, remove, and outcome)

**API handlers** must emit at minimum:
- Request duration (histogram by endpoint and status class)
- Request count (counter by endpoint and status class)

Metric naming must follow Prometheus conventions: `snake_case`, service-prefixed, with unit suffixes where applicable (e.g. `_seconds`, `_total`). Shared metric definitions belong in the core library.

#### 9.4.1 Metric Labels and Cardinality

Metric labels must be used only for dimensions with bounded, low cardinality. Do not label metrics with unbounded or per-instance values such as resource IDs, user IDs, request IDs, or free-form user input unless the dimension is explicitly required and its cardinality is known to be small.

Where a metric is intentionally split by a named platform value, such as a server region, emit both the stable identifier and the display name as labels. The identifier is authoritative and is used for alerting; the name is for human-readable dashboards and chart legends.

Use `_id` and `_name` suffixes to distinguish these labels, for example `region_id` and `region_name`.

---

## 10. Security

The platform enforces three tiers of access control, each with a distinct authentication mechanism:

| Tier | Principals | Authentication |
|---|---|---|
| Public | End users and tooling | OIDC tokens. Access only via service REST APIs. |
| Service | Platform services | mTLS exclusively. Certificate CN maps to RBAC role. |
| Platform | Cloud provider APIs | Accessed only by the Region service via Kubernetes Secret credentials. |

RBAC as defined in [section 4.3](#43-rbac) is the enforcement mechanism for Public and Service tier access. No principal in any tier may access resources outside their permitted scope as defined by the ACL.

### 10.1 Platform Security Invariants

The following invariants apply unconditionally to all platform components. Any implementation that violates them is a defect, not a design trade-off.

- **No privilege escalation** — no operation may grant a principal more access than they already hold. Impersonation can only narrow a user's effective permissions (via ACL intersection with the service's own ACL); it can never expand them. See [section 4.6.1](#acl-intersection-under-impersonation).
- **Principle of least authority** — each actor (service or user) operates with only the permissions required for the current operation. Services must not accumulate permissions beyond their functional scope, and must not forward a user's full ACL when acting as a proxy.
- **Scope confinement** — access granted at a narrower scope (project, organisation) does not implicitly confer access at a broader scope. The scope hierarchy is strictly one-directional: broader grants cover narrower scopes, never the reverse. See [section 4.3](#43-rbac).
- **Single enforcement point** — all access decisions are made against the ACL returned by the identity service. There is no local policy evaluation in individual services. Duplicating or caching access logic outside the ACL endpoint is a defect.
- **Immutable attribution** — the principal recorded on a resource at creation time cannot be changed. Re-attribution is not permitted. See [section 4.4](#44-principals-and-proxies).

---

## 11. Core Library

Before implementing any pattern in a platform service, check the core library. Reimplementing existing patterns is a defect.

The core library is not a deployed service. It is a shared Go library that provides the canonical implementations of all platform design patterns — including the work queue, saga, quota reservation, reference management, middleware stack, error types, and all RBAC helpers. If a pattern is described in this specification, its reference implementation is in the core library.

---

## 12. Glossary

| Term | Definition |
|---|---|
| ACL | Computed set of endpoint scopes and CRUD permissions for a principal |
| CloudIdentity | Cloud project/user/role provisioned by the Region service for a consuming service |
| ClusterManager | Isolated Cluster API instance; multi-tenant cluster lifecycle management |
| CN | X.509 Common Name — service identity for mTLS; maps to an RBAC role |
| CRD | Custom Resource Definition — Kubernetes extension for all platform resource types |
| DAG | Directed Acyclic Graph — structure of resource containment and access scope hierarchy |
| Data Boundary | Each service exclusively owns its data; others access only via versioned API |
| DELETING | Deletion state: tombstoned, blocks while references exist |
| DRAINING | Deletion state: references cleared, cleaning owned children |
| FINALIZING | Deletion state: children gone, releasing references on parent resources |
| DELETED | Deletion state: all references released, resource garbage collected |
| `infra-manager-service` | RBAC role: read on regions, read/delete on identities and physicalnetworks |
| mTLS | Mutual TLS — both sides present certificates; all service-to-service auth |
| OIDC | OpenID Connect — end-user authentication protocol |
| Containment Edge | Edge carrying scope propagation and forward deletion propagation; resource cannot exist outside the scope of its container |
| Principal | Originating end-user responsible for a resource; determines quota and billing |
| Consumption Edge | Edge carrying reverse deletion propagation (blocking) and co-location; resource uses another it does not contain |
| Dependency Edge | Edge carrying reverse deletion propagation (triggering) and co-location; deletion of target triggers deletion of source |
| `ErrYield` | Sentinel error signalling expected transient blocking; triggers fixed-period requeue, not exponential backoff |
| Project | Organisational workspace; root scope for all user resources |
| Protected Role | RBAC role hidden from public API; grantable only via Helm values |
| Proxy | A service acting on behalf of a principal to provision resources |
| Reference | Named dependency claim on a resource; blocks deletion until removed |
| Region | A registered cloud provider instance |
| Status Propagation | Edge property: state change on the target enqueues the source for reconciliation; source re-derives its own status from aggregate state of connected targets |
| Core Library | Shared Go library of canonical platform patterns; the reference implementation of all platform design patterns |
| `GenerateResourceReference` | Core library function producing a deterministic reference string: `{resource}.{group}/{uuid}`. The canonical key for allocations and resource references. |
| UUID v4 | Randomly generated universally unique identifier. Standard format for all resource names on the platform. |
| UUID v5 | Deterministically generated UUID (RFC 4122). Used only for index resources whose sole purpose is uniqueness enforcement. Never used for resources carrying billing, quota, or audit identity. See Appendix A.1. |
| `unikorn-client-issuer` | Root CA for all inter-service mTLS |
| Delegated Principal | A user principal propagated by a proxy service across an internal API boundary. The transport is mTLS; the actor for RBAC, quota, billing, and audit is the user, not the proxy. See [section 4.5.1](#451-api-authentication-classifications). |
| `x-internal: true` | OpenAPI operation extension marking an endpoint as internal. Middleware enforces mTLS client certificate authentication and rejects OAuth2 bearer tokens. |
| `x-principal: delegated` | OpenAPI operation extension used with `x-internal: true`. Middleware asserts a user principal must be present in the request context. RBAC resolves against the user principal, not the mTLS CN. |

---

## Appendix A: Known Issues

### A.1 TOCTOU Race — Resource Name Uniqueness

**Fix** (applies to all services with per-network hostname uniqueness constraints)

Introduce a `HostnameIndex` CRD per service (e.g. `ServerHostnameIndex` in `uni-region`, `InstanceHostnameIndex` in `uni-compute`). Its sole purpose is to hold a uniqueness slot — it is a mutex, not a managed resource, and carries no billing or identity significance.

The index resource name is derived as UUID v5, seeded from a fixed platform namespace UUID constant (defined in the core library) and the string `networkID+"/"+hostname`. Two concurrent creates with the same inputs produce the same UUID v5 and collide natively at the storage layer. One receives a native `409`. No application-level race detection is required.

The resource name (Server, ComputeInstance, etc.) remains UUID v4 — random, unique across all lifetimes, unambiguous for billing, quota, and audit.

The index entry is created before the resource. If index creation returns `409`, the handler returns `409` to the caller and no resource is created. If resource creation subsequently fails, the saga compensating transaction deletes the index entry.

The resource is set as the Kubernetes owner of its index entry at create time. When the resource is deleted, Kubernetes garbage collects the index entry automatically via owner references.

When the index entry is garbage collected, the UUID v5 slot is free. A new resource with the same hostname on the same network gets a fresh UUID v4 and creates a new index entry under the same UUID v5 name. Previous billing and audit records point at the old UUID v4 — unambiguous, no collision across lifetimes.

`isServerNameInUse()`, `isInstanceNameInUse()`, and any equivalent list-then-check functions become redundant and must be deleted.

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
