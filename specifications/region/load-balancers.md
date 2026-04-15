# Region Load Balancers v2

This document defines the public `uni-region` v2 load balancer API. It is the normative public abstraction for Layer 4 load balancing. Provider-specific mapping for the initial OpenStack Octavia Amphora implementation is documented in [OpenStack Load Balancers](../providers/openstack/load-balancers.md).

The standard Region v2 resource envelope and the platform-wide rules in [SPECIFICATION.md](../../SPECIFICATION.md) still apply. This document defines the load-balancer-specific API surface, schema, validation rules, and cross-service semantics.

## Changelog

| Version | Date | Notes |
|---|---|---|
| v0.2 | 2026-04-15 | Align network deletion semantics with owner-based foreground cascade while keeping network status propagation. |
| v0.1 | 2026-04-13 | Initial version |

## Initial Scope

The first public abstraction is intentionally narrow:

- Layer 4 only
- Listener protocols limited to `tcp` and `udp`
- Listener-scoped `allowedCidrs`
- Top-level `publicIP`
- Optional create-time selection of the primary internal VIP
- Backend members specified by IPv4 address
- Per-member backend port
- Optional transport-level health checks only
- Optional TCP-only `proxyProtocolV2`
- A single TCP idle-timeout control
- IPv4 only

The following are explicitly not part of v1:

- HTTP and HTTPS listeners
- Terminated TLS
- L7 routing or policies
- Weighted or backup members
- Session persistence
- Alternate monitor IP or port
- Non-L4 health check types
- Additional Amphora timeout knobs beyond `idleTimeoutSeconds`
- IPv6 VIPs or members
- Mixed IPv4 and IPv6 members
- Additional VIPs
- VIP changes after create
- Listener connection limits

Terminated TLS is intentionally excluded in v1 so Kubernetes and CCM-managed workloads retain end-to-end TLS and, where used, client-certificate visibility at the workload. Regions that expose this API are assumed to provide Octavia/Amphora support; unsupported regions are out of scope rather than modeled as a discoverable per-region capability flag.

## API Surface

Add the following Region v2 endpoints:

- `GET /api/v2/loadbalancers`
- `POST /api/v2/loadbalancers`
- `GET /api/v2/loadbalancers/{loadBalancerID}`
- `PUT /api/v2/loadbalancers/{loadBalancerID}`
- `DELETE /api/v2/loadbalancers/{loadBalancerID}`

All endpoints are **Public** (OAuth2 bearer token, no OpenAPI extension required). RBAC is enforced against the derived user principal per [SPECIFICATION.md §7.2.1](../../SPECIFICATION.md).

All write operations are asynchronous and return `202 Accepted`. Callers must poll `GET /api/v2/loadbalancers/{loadBalancerID}` and read `status.conditions` to determine whether provisioning or deletion has completed. There is no `GET /api/v2/loadbalancers/{loadBalancerID}/status`. `DELETE` on a resource that already has a deletion timestamp set returns `202` without re-triggering deletion.

List filters follow existing Region v2 conventions:

- `organizationID`
- `projectID`
- `regionID`
- `networkID`
- `tag`

The `tag` filter matches the standard user-facing resource tags exposed at `metadata.tags` in the shared UNI resource envelope and is evaluated post-list per [SPECIFICATION.md §7.8](../../SPECIFICATION.md). This document does not add a load-balancer-specific public `tags` field under `spec`. The backing Region CRD may persist tags at `spec.tags` as an internal implementation detail; that does not change the public API contract.

List results are sorted stably by name for deterministic ordering per [SPECIFICATION.md §7.8](../../SPECIFICATION.md).

RBAC adds the new scope:

- `region:loadbalancers:v2`

## Public Types

Add the following public OpenAPI types and generated clients:

- `LoadBalancerV2Create`
- `LoadBalancerV2CreateSpec`
- `LoadBalancerV2Update`
- `LoadBalancerV2Spec`
- `LoadBalancerV2Read`
- `LoadBalancersV2Read`
- `LoadBalancerV2Status`
- `LoadBalancerListenerV2`
- `LoadBalancerPoolV2`
- `LoadBalancerMemberV2`
- `LoadBalancerHealthCheckV2`

Add the list query parameter type:

- `GetApiV2LoadbalancersParams`

Add the path parameter:

- `loadBalancerIDParameter`

## Resource Model

The standard Region v2 envelope applies. This section defines the `spec` and `status` payloads specific to load balancers.

### Create Spec

`LoadBalancerV2CreateSpec` contains:

| Field | Required | Description |
|---|---|---|
| `organizationId` | yes | Owning organization. Immutable after create. |
| `projectId` | yes | Owning project. Immutable after create. |
| `regionId` | yes | Target region. Immutable after create. |
| `networkId` | yes | Tenant network used for the VIP subnet and backend member resolution. Immutable after create. |
| `vipAddress` | no | Requested primary internal IPv4 VIP on the load balancer tenant network. Omit to preserve current provider auto-allocation behavior. |
| `publicIP` | no | When `true`, request a public IPv4 address for the VIP. |
| `listeners` | yes | Listener definitions. |

### Mutable Spec

`LoadBalancerV2Spec` contains:

| Field | Required | Description |
|---|---|---|
| `publicIP` | no | Mutable after create. |
| `listeners` | yes | Listener and pool configuration. |

### Status

`LoadBalancerV2Status` contains:

| Field | Required | Description |
|---|---|---|
| `regionId` | yes | Region the load balancer is running in. |
| `networkId` | yes | Tenant network backing the VIP and member reachability. |
| `vipAddress` | no | Actual VIP address allocated for the load balancer. |
| `publicIP` | no | Public IPv4 address attached to the VIP when enabled. |
| `conditions` | yes | Standard platform `status.conditions`. |

Rules:

- `organizationId`, `projectId`, `regionId`, and `networkId` are immutable after create.
- `vipAddress` is optional on create. When omitted, the provider allocates the private VIP exactly as it does today.
- `vipAddress` is create-only. Because it is absent from `LoadBalancerV2Spec`, `PUT` cannot create, change, or clear the requested VIP.
- `publicIP` is mutable.
- `networkId` is the required backend network. All member addresses must be reachable on this network.
- `spec.publicIP` requests a public VIP address and `status.publicIP` reports the allocated public IPv4. This intentionally matches the existing instance API naming convention.
- `status.vipAddress` reports the actual allocated VIP for the load balancer.
- If create succeeds with `spec.vipAddress` set, `status.vipAddress` must equal the requested value.
- `status.vipAddress` is stable for the lifetime of the load balancer. Once allocated, the VIP does not change.
- `Available=True` must not be reported until all of the following hold: `status.vipAddress` is populated, at least one effective backend exists, and (when `spec.vipAddress` was set on create) `status.vipAddress` equals the requested value.
- The public status surface is provider-agnostic. Provider IDs, per-listener operating status, per-member health, statistics, the algorithm, the status tree, and driver names are not exposed in v1.

## Listener Schema

Each listener is represented by `LoadBalancerListenerV2`:

| Field | Required | Description |
|---|---|---|
| `name` | yes | Stable listener identity, unique within the load balancer. |
| `protocol` | yes | `tcp` or `udp`. Immutable after create. |
| `port` | yes | Frontend port number. Immutable after create. |
| `allowedCidrs` | no | IPv4 CIDRs allowed to reach this listener. |
| `idleTimeoutSeconds` | no | TCP-only idle timeout override. |
| `pool` | yes | Exactly one backend pool for this listener. |

Listener rules:

- `name` is the stable reconciliation identity for the listener.
- `protocol` and `port` are immutable after create. On update, the handler must verify that `protocol` and `port` match the existing listener with the same `name`. A mismatch is rejected with `422 unprocessable_content`. To change a listener's protocol or port, remove the listener and add a new one.
- Renaming a listener is treated as delete-and-create, not an in-place rename.
- `name` must be 1 to 63 characters, use only ASCII alphanumerics and `-`, start with an alphanumeric, end with an alphanumeric, and be unique within the load balancer.
- `(protocol, port)` must be unique within the load balancer.
- `port` must be an integer in the range `1..65535`.
- `allowedCidrs` uses OR semantics.
- Omitted or empty `allowedCidrs` means unrestricted ingress for that listener on both create and update.
- Every `allowedCidrs` entry must be a valid IPv4 CIDR.
- `idleTimeoutSeconds` is valid only for `tcp`.
- `idleTimeoutSeconds` must be an integer in the range `1..86400` (1 second to 24 hours).
- Omitted `idleTimeoutSeconds` means provider default.

## Pool Schema

Each listener owns exactly one `LoadBalancerPoolV2`:

| Field | Required | Description |
|---|---|---|
| `proxyProtocolV2` | no | TCP-only backend PROXY protocol version 2 toggle. |
| `members` | yes | Backend members. The field is required but may be empty. |
| `healthCheck` | no | Optional transport-level health check. |

Pool rules:

- Exactly one pool exists per listener.
- `proxyProtocolV2` is valid only for `tcp` listeners.
- Omitted `proxyProtocolV2` defaults to `false`.
- `proxyProtocolV2=true` means the backend connection is wrapped in PROXY protocol version 2.
- `members` may be `[]` on create or update.
- A load balancer with zero members remains present and still receives a VIP. See Status rules for `Available` preconditions.
- There is no public `algorithm` field.
- The implementation fixes the hidden algorithm to `ROUND_ROBIN`.

## Member Schema

Members are specified by IPv4 address. This remains IP-only in v1 because Octavia pool membership is directly IP-based, and a server-reference variant would introduce compute-resource deletion semantics that conflict with expected CCM/CAPI scale-down flows.

Each `LoadBalancerMemberV2` contains:

| Field | Required | Description |
|---|---|---|
| `address` | yes | IPv4 address of the backend member. |
| `port` | yes | Backend port used for this member. |

Member rules:

- `address` is required and must be a valid IPv4 address (not a CIDR, not a hostname).
- `port` is required and must be an integer in the range `1..65535`.
- Duplicate `(address, port)` pairs in one pool are rejected.
- The same `address` may appear in different pools if the resulting `(address, port)` backends remain distinct.
- The same `address` may appear multiple times in one pool with different `port` values.

## Health Check Schema

`LoadBalancerHealthCheckV2` is optional and contains:

| Field | Required | Description |
|---|---|---|
| `intervalSeconds` | no | Probe interval. Default `10` when enabled. |
| `timeoutSeconds` | no | Per-probe timeout. Default `5` when enabled. |
| `healthyThreshold` | no | Successes required to mark healthy. Default `2` when enabled. |
| `unhealthyThreshold` | no | Failures required to mark unhealthy. Default `2` when enabled. |

Health-check rules:

- Omitted or `null` means disabled.
- `{}` means enabled with defaults.
- There is no `enabled` field.
- There is no `protocol` field.
- There is no `port` field.
- There are no HTTP, HTTPS, TLS, or other L7-specific health-check fields.
- `intervalSeconds` and `timeoutSeconds` must be integers in the range `1..300`.
- `healthyThreshold` and `unhealthyThreshold` must be integers in the range `1..10`.
- `timeoutSeconds` must be less than `intervalSeconds` when a health check is present.
- Health-check add, remove, and parameter updates are supported in place.
- Health checks always target the member's configured backend port using transport semantics derived from the listener protocol and pool settings.

## Validation and Error Semantics

The request body is first validated against the OpenAPI schema. Schema or field-format failures, including malformed `vipAddress`, should return `400 invalid_request`. Cross-field or cross-resource semantic failures should return `422 unprocessable_content`.

Create and update validation must enforce all of the following:

- `organizationId`, `projectId`, `regionId`, and `networkId` remain immutable after create.
- `vipAddress`, when present in `LoadBalancerV2CreateSpec`, must be a valid IPv4 address.
- `vipAddress` must not be accepted on `PUT`. Because `LoadBalancerV2Spec` excludes the field, any `PUT` body containing `vipAddress` is schema-invalid and returns `400 invalid_request`.
- The referenced `networkId` must share the same organisational scope root as the load balancer (co-location constraint per [SPECIFICATION.md §5.10](../../SPECIFICATION.md)). Reject with `422 unprocessable_content` if not.
- On update, each listener's `protocol` and `port` must match the existing listener with the same `name`. A mismatch is rejected with `422 unprocessable_content`.
- Listener names satisfy the documented format and uniqueness rules.
- Listener ports and member ports are integers in the range `1..65535`.
- `idleTimeoutSeconds`, when present, is an integer in the range `1..86400`.
- Health-check intervals and timeouts are integers in the range `1..300`.
- Health-check thresholds are integers in the range `1..10`.
- `proxyProtocolV2=true` is rejected for `udp`.
- `idleTimeoutSeconds` is rejected for `udp`.
- `timeoutSeconds` must be less than `intervalSeconds` when a health check is present.
- Duplicate listener names are rejected.
- Duplicate `(protocol, port)` listeners are rejected.
- Every member `address` must be a valid IPv4 address.
- Duplicate `(address, port)` members in the same pool are rejected.
- Member addresses are not validated for reachability at submission time. Network-layer reachability is the caller's responsibility.

If `vipAddress` is set, reconcile must either allocate that exact VIP or fail the resource. It must never substitute a different private VIP when a specific internal VIP was requested.

Because writes are asynchronous, a create request with a syntactically valid but unusable `vipAddress` still returns `202 Accepted` if it passes request-time validation. If allocation later fails because the requested address is off-network, already in use, unavailable, or otherwise rejected by the provider, reconcile sets `Available=False` with reason `ConditionReasonErrored`. `status.vipAddress` remains empty until the exact requested VIP is actually allocated.

Update operations use optimistic locking via `MergeFromWithOptimisticLock` per [SPECIFICATION.md §7.3](../../SPECIFICATION.md). If the resource version has advanced since the client last read the resource, the write is rejected with `409 conflict`.

Preferred response behavior:

| Condition | Preferred response |
|---|---|
| Invalid `vipAddress` syntax, invalid IPv4 CIDR syntax in `allowedCidrs`, invalid listener name format, invalid member `address`, or out-of-range numeric field values | `400 invalid_request` |
| Authentication failure or expired token | `401 access_denied` |
| Authenticated principal lacks permission, or quota exhausted | `403 forbidden` |
| Resource not found on GET, PUT, or DELETE | `404 not_found` |
| Write conflict on PUT (resource version advanced) | `409 conflict` |
| `vipAddress` present in a `PUT` body | `400 invalid_request` |
| Listener `protocol` or `port` changed on update (mismatch with existing listener of the same `name`) | `422 unprocessable_content` |
| Unsupported field combinations such as `proxyProtocolV2=true` on `udp` or `idleTimeoutSeconds` on `udp` | `422 unprocessable_content` |
| Co-location constraint violation (`networkId` not in same organisational scope) | `422 unprocessable_content` |
| `timeoutSeconds >= intervalSeconds` | `422 unprocessable_content` |
| Duplicate listener names, duplicate `(protocol, port)` listeners, or duplicate `(address, port)` members in one pool | `422 unprocessable_content` |
| Unexpected internal error | `500 server_error` |

All error responses follow the standard platform error body format (`error`, `error_description`, `trace_id`) per [SPECIFICATION.md §7.9](../../SPECIFICATION.md). The `PUT` endpoint must advertise `409` in the OpenAPI spec per [SPECIFICATION.md §7.3](../../SPECIFICATION.md).

## Quota Semantics

Per [SPECIFICATION.md §7.6](../../SPECIFICATION.md), v1 requires quota enforcement for both load balancer count and public IP allocation. The quota kind for public IPs is `publicips`, which is a shared quota defined in `uni-identity` (default: 5) and consumed by all services that allocate public IPs (compute instances, load balancers, etc.). Requested internal VIPs do not introduce a new quota kind and do not change existing `loadbalancers` or `publicips` accounting.

- **Create**: the saga reserves `1` `loadbalancers` quota unit keyed by the load balancer resource UUID. If `spec.publicIP=true`, the saga also reserves `1` `publicips` unit keyed by the same UUID. If either quota is exhausted, the handler returns `403 forbidden`.
- Successful provisioning promotes both reservations to committed allocations.
- **Update**: if `publicIP` changes from `false` to `true`, the handler reserves `1` `publicips` unit before patching the CRD. If quota is exhausted, the handler returns `403 forbidden` without modifying the resource. The reservation is promoted to a committed allocation by the controller on successful provisioning. If the CRD patch fails (e.g., optimistic lock conflict), the handler releases the reservation as saga compensation. If `publicIP` changes from `true` to `false`, the controller releases the `publicips` unit during deprovisioning of the floating IP. Changes to `publicIP` do not affect the `loadbalancers` quota.
- **Delete**: releasing the committed allocation covers both `loadbalancers` and `publicips` (whichever was committed) before the controller removes its finalizer.

## Resource Graph

### Edge Declarations

Per [SPECIFICATION.md §5.8](../../SPECIFICATION.md) and [§5.10](../../SPECIFICATION.md), the LoadBalancer resource type declares separate lifecycle and status relationships for the same resource pair because deletion ownership and status observation point in opposite directions:

| Relationship | Properties |
|---|---|
| Network → LoadBalancer | Forward Deletion Propagation via Kubernetes owner reference with blocking owner deletion / foreground cascading semantics |
| LoadBalancer → Network | Status Propagation Upward + Co-location Required |

Both relationships are intra-service (the Region service owns both resource types). No long-lived reference API hold is placed on the Network during steady-state load-balancer lifetime.

### Network Ownership and Status Observation

- `networkId` is immutable after create.
- On create, the handler validates the referenced `Network` and creates the backing `LoadBalancer` resource with that `Network` as its Kubernetes owner. Because `networkId` cannot change later, this owner relationship is fixed for the lifetime of the load balancer.
- The owner reference uses blocking owner deletion / foreground cascading semantics. When the `Network` is deleted, Kubernetes tombstones the owned `LoadBalancer` as part of the same intra-service delete flow.
- Standalone `DELETE` of a load balancer is unchanged.
- During a network delete, the load balancer controller does not independently synthesize deletion from the `Network` deletion timestamp. It reconciles the already-deleting `LoadBalancer`, performs normal provider deprovisioning, releases quota state, and removes its finalizer.
- The controller registers a Kubernetes watch on the `Network` resource type and directly `Get`s the referenced `Network` during reconcile for status propagation only. These reads are not used to synthesize load-balancer deletion from a network tombstone.
- When the `Network` status changes, the controller re-derives the load balancer's status from the aggregate state of connected targets (status propagation upward per [SPECIFICATION.md §5.8](../../SPECIFICATION.md)).
- If the `LoadBalancer` already has a deletion timestamp, a missing or deleting `Network` is expected during owner-driven cascade and must not be treated as a separate dependency-triggered delete path.
- If the `LoadBalancer` is not deleting and the observed `Network` is unavailable, the load balancer reflects this in `Available=False`.

### Member Address Semantics

- Members are specified by IPv4 address. There is no edge relationship or cross-service dependency on compute instances.
- This aligns the public API with Octavia's native member model and avoids introducing a server-reference abstraction that would incorrectly block instance deletion during CCM-managed or Cluster API scale-down.
- The load balancer does not validate member address reachability at submission time.
- With `healthCheck` enabled, provider health monitoring will detect and remove unreachable backends from rotation.
- Without `healthCheck`, traffic may be attempted to unreachable backends until the client updates the member list.

### Owned Resources

[SPECIFICATION.md §2.1](../../SPECIFICATION.md) includes `LoadBalancer` in the Region service's owned resources table.

## Handler Semantics

The standard Region v2 handler layer responsibilities ([SPECIFICATION.md §7.2](../../SPECIFICATION.md)) apply. Load-balancer-specific notes:

- **Create**: the handler validates the referenced `Network`, derives labels and annotations via `conversion.NewObjectMetadata`, and never accepts labels or annotations from the request body. It uses a saga with soft reservation steps for `1` `loadbalancers` quota unit and, when `spec.publicIP=true`, `1` `publicips` quota unit per [SPECIFICATION.md §7.5, §7.6](../../SPECIFICATION.md), then creates the `LoadBalancer` CRD as the terminal write with the referenced `Network` set as Kubernetes owner using blocking owner deletion / foreground cascading semantics. If `spec.vipAddress` is present, the handler persists it as a distinct immutable internal field (for example `requestedVipAddress`) on the backing resource so reconcile can replay the same provider request idempotently across retries and restarts. If either quota is exhausted, it returns `403 forbidden`.
- **Update**: the handler reads the current resource, re-derives labels and annotations, and patches with optimistic locking via `MergeFromWithOptimisticLock`. A concurrent modification returns `409 conflict`. `vipAddress` is intentionally absent from `LoadBalancerV2Spec`; any `PUT` body containing it fails schema validation with `400 invalid_request`. If `publicIP` toggles from `false` to `true`, the handler reserves `1` `publicips` unit before patching the CRD and returns `403 forbidden` if exhausted; the reservation is released as compensation if the CRD patch fails. If `publicIP` toggles from `true` to `false`, the `publicips` unit is released by the controller during floating IP deprovisioning.
- **Delete**: the handler verifies `DeletionTimestamp` is nil before issuing the delete. If already set, it returns `202` idempotently.

All POST, PUT, and DELETE operations emit audit log entries per [SPECIFICATION.md §9.2](../../SPECIFICATION.md).

## Controller Lifecycle

### Finalizer

The controller adds its finalizer to the `LoadBalancer` resource before performing any provisioning work. On deletion, whether initiated directly or via owner-driven `Network` cascade, the controller detects the load balancer deletion timestamp, retries silently while owned children or references exist, runs full deprovisioning (including all provider resources), releases any committed quota allocation or surviving reservation, and removes the finalizer only after deprovisioning completes per [SPECIFICATION.md §8.5](../../SPECIFICATION.md).

### Deadlock Detection

On every reconcile pass of a load balancer in `DELETING` state, the controller checks the age of the deletion timestamp. If the timestamp exceeds 10 minutes and inbound references are still present, the controller emits a structured zap log entry with: resource type, resource ID, deletion timestamp, elapsed duration, and the full set of blocking reference strings as discrete structured fields per [SPECIFICATION.md §8.8](../../SPECIFICATION.md).

## Observability

The load balancer API handlers and controller follow the platform observability requirements in [SPECIFICATION.md §9](../../SPECIFICATION.md):

- **Structured logging**: the controller logs every reconcile pass, state transition, reference operation, and requeue decision with resource type and resource ID as structured zap fields. API handlers log on non-2xx responses only.
- **Audit logging**: all authenticated, scoped, state-mutating API operations (POST, PUT, DELETE) emit `msg: audit` entries at info level with the required fields (actor, operation, scope, resource, result).
- **Distributed tracing**: a span is created for every inbound API request and every controller reconcile pass. The trace ID is included in all non-2xx error responses.
- **Metrics**: the controller emits reconcile duration (histogram), reconcile outcome (counter), work queue depth (gauge), and reference operation count (counter). API handlers emit request duration (histogram) and request count (counter), both by endpoint and status class.

## Defaults and Derived Behavior

The initial public behavior is:

- `publicIP` is top-level on the load balancer spec.
- Omitted `vipAddress` preserves current provider auto-allocation behavior for the primary internal VIP.
- `vipAddress`, when supplied on create, selects only the primary internal VIP and remains immutable after create.
- `allowedCidrs` is listener-scoped.
- `proxyProtocolV2` is v2-only, TCP-only, and defaults to `false`.
- `healthCheck` object presence means enabled.
- Health-check omission means disabled.
- `healthCheck: {}` means enabled with defaults.
- Health-check add, remove, and parameter updates are supported in place.
- Omitted `idleTimeoutSeconds` uses provider defaults.
- Zero members keep the load balancer present but not yet available.
- The implementation is IPv4-only.

Provider-specific protocol, health-monitor, timeout, and live-discovery mapping is defined in [OpenStack Load Balancers](../providers/openstack/load-balancers.md).

## Open Questions

- Should there be maximum limits on the number of listeners, members per pool, or `allowedCidrs` entries per listener? Octavia may impose its own limits that should be surfaced.
- When a load balancer has zero members and Octavia reports `ACTIVE` with a VIP, the condition is `ConditionReasonProvisioning`. Should the platform condition vocabulary be extended to distinguish "provisioned but not operational" from "provisioning in progress"?
