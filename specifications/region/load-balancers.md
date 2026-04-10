# Region Load Balancers v2

This document defines the public `uni-region` v2 load balancer API. It is the normative public abstraction for Layer 4 load balancing. Provider-specific mapping for the initial OpenStack Octavia Amphora implementation is documented in [OpenStack Load Balancers](../providers/openstack/load-balancers.md).

The standard Region v2 resource envelope and the platform-wide rules in [SPECIFICATION.md](../../SPECIFICATION.md) still apply. This document defines the load-balancer-specific API surface, schema, validation rules, and cross-service semantics.

## Changelog

| Version | Date | Notes |
|---|---|---|
| v0.2 | 2026-04-10 | Review update: empty member sets, dependency-triggered deletion, stale-member tolerance, and clarified validation/status semantics |
| v0.1 | 2026-04-10 | Initial draft |

## Initial Scope

The first public abstraction is intentionally narrow:

- Layer 4 only
- Listener protocols limited to `tcp` and `udp`
- Listener-scoped `allowedCidrs`
- Top-level `publicIP`
- Backend members limited to `uni-compute` instances
- Per-member backend port
- Optional transport-level health checks only
- Optional TCP-only `proxyProtocolV2`
- A single TCP idle-timeout control
- IPv4 only

The following are explicitly not part of v1:

- Raw IP-address members
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
- User-specified VIP addresses
- Kubernetes `spec.loadBalancerIP`-style workflows
- Listener connection limits

## API Surface

Add the following Region v2 endpoints:

- `GET /api/v2/loadbalancers`
- `POST /api/v2/loadbalancers`
- `GET /api/v2/loadbalancers/{loadBalancerID}`
- `PUT /api/v2/loadbalancers/{loadBalancerID}`
- `DELETE /api/v2/loadbalancers/{loadBalancerID}`

All write operations are asynchronous and return `202 Accepted`. Callers must poll `GET /api/v2/loadbalancers/{loadBalancerID}` and read `status.conditions` to determine whether provisioning or deletion has completed. There is no `GET /api/v2/loadbalancers/{loadBalancerID}/status`.

List filters follow existing Region v2 conventions:

- `organizationID`
- `projectID`
- `regionID`
- `networkID`
- `tag`

The `tag` filter applies generic resource metadata tags inherited from the standard Region v2 envelope. This document does not add a load-balancer-specific `tags` field under `spec`.

RBAC adds the new scope:

- `region:loadbalancers:v2`

Provider capability discovery adds:

- `region.features.loadBalancers: bool`

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
| `vipAddress` | yes | VIP address allocated for the load balancer. |
| `publicIP` | no | Public IPv4 address attached to the VIP when enabled. |
| `conditions` | yes | Standard platform `status.conditions`. |

Rules:

- `organizationId`, `projectId`, `regionId`, and `networkId` are immutable after create.
- `publicIP` is mutable.
- `networkId` is the required backend network for all referenced compute instances.
- `spec.publicIP` requests a public VIP address and `status.publicIP` reports the allocated public IPv4. This intentionally matches the existing instance API naming convention.
- There is no `vipAddress` request field in create or update. Callers cannot request a specific VIP or private address in v1, so Kubernetes `spec.loadBalancerIP`-style workflows are unsupported.
- The public status surface is provider-agnostic. Provider IDs, per-listener operating status, per-member health, statistics, the algorithm, the status tree, and driver names are not exposed in v1.

## Listener Schema

Each listener is represented by `LoadBalancerListenerV2`:

| Field | Required | Description |
|---|---|---|
| `name` | yes | Stable listener identity, unique within the load balancer. |
| `protocol` | yes | `tcp` or `udp`. |
| `port` | yes | Frontend port number. |
| `allowedCidrs` | no | IPv4 CIDRs allowed to reach this listener. |
| `idleTimeoutSeconds` | no | TCP-only idle timeout override. |
| `pool` | yes | Exactly one backend pool for this listener. |

Listener rules:

- `name` is the stable reconciliation identity for the listener.
- Renaming a listener is treated as delete-and-create, not an in-place rename.
- `name` must be 1 to 63 characters, use only ASCII alphanumerics and `-`, start with an alphanumeric, end with an alphanumeric, and be unique within the load balancer.
- `(protocol, port)` must be unique within the load balancer.
- `port` must be an integer in the range `1..65535`.
- `allowedCidrs` uses OR semantics.
- Omitted or empty `allowedCidrs` means unrestricted ingress for that listener on both create and update.
- Every `allowedCidrs` entry must be a valid IPv4 CIDR.
- `idleTimeoutSeconds` is valid only for `tcp`.
- `idleTimeoutSeconds` must be an integer greater than or equal to `1`.
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
- A load balancer with zero effective backends, meaning no member currently resolves to a live in-scope instance with a usable private IP, remains present and still receives a VIP, but does not report `Available=True` until at least one backend becomes effective.
- There is no public `algorithm` field.
- The implementation fixes the hidden algorithm to `ROUND_ROBIN`.

## Member Schema

Members are limited to `uni-compute` instances in v1.

Each `LoadBalancerMemberV2` contains:

| Field | Required | Description |
|---|---|---|
| `instanceId` | yes | Referenced compute instance. |
| `port` | yes | Backend port used for this member. |

Member rules:

- `instanceId` is required.
- `port` is required and must be an integer in the range `1..65535`.
- Duplicate `instanceId` values in one pool are rejected.
- The same `instanceId` may appear in different pools if the resulting backends remain distinct.
- Duplicate resolved `(privateIP, port)` backends are rejected once instance IPs are known.
- Raw IP-address members are unsupported in v1.

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
- `intervalSeconds` and `timeoutSeconds` must be integers greater than or equal to `1`.
- `healthyThreshold` and `unhealthyThreshold` must be integers in the range `1..10`.
- `timeoutSeconds` must be less than `intervalSeconds` when a health check is present.
- Health-check add, remove, and parameter updates are supported in place.
- Health checks always target the member's configured backend port using transport semantics derived from the listener protocol and pool settings.

## Validation and Error Semantics

The request body is first validated against the OpenAPI schema. Schema or field-format failures should return `400 invalid_request`. Cross-field or cross-resource semantic failures should return `422 unprocessable_content`.

Create and update validation must enforce all of the following:

- `organizationId`, `projectId`, `regionId`, and `networkId` remain immutable after create.
- Listener names satisfy the documented format and uniqueness rules.
- Listener ports and member ports are integers in the range `1..65535`.
- `idleTimeoutSeconds`, when present, is an integer greater than or equal to `1`.
- Health-check intervals and timeouts are integers greater than or equal to `1`.
- Health-check thresholds are integers in the range `1..10`.
- `proxyProtocolV2=true` is rejected for `udp`.
- `idleTimeoutSeconds` is rejected for `udp`.
- `timeoutSeconds` must be less than `intervalSeconds` when a health check is present.
- Duplicate listener names are rejected.
- Duplicate `(protocol, port)` listeners are rejected.
- Duplicate `instanceId` members in the same pool are rejected.
- Duplicate resolved `(privateIP, port)` backends are rejected.
- On create, every supplied `instanceId` resolves through the Compute v2 API and belongs to the same organization, project, region, and network as the load balancer.
- A referenced compute instance may exist before private IP assignment. In that case the load balancer remains provisioning until the address resolves.
- On update, every newly added or modified member must resolve and belong to the same organization, project, region, and network as the load balancer.
- On update, unchanged stale members whose instances are now `DELETING` or already gone are tolerated temporarily so clients can perform unrelated updates and then remove or replace them.

Preferred response behavior:

| Condition | Preferred response |
|---|---|
| Invalid IPv4 CIDR syntax in `allowedCidrs`, invalid listener name format, or out-of-range numeric field values | `400 invalid_request` |
| Unsupported field combinations such as `proxyProtocolV2=true` on `udp` or `idleTimeoutSeconds` on `udp` | `422 unprocessable_content` |
| `timeoutSeconds >= intervalSeconds` | `422 unprocessable_content` |
| Duplicate listener names, duplicate `(protocol, port)` listeners, duplicate `instanceId` members in one pool, or duplicate resolved `(privateIP, port)` backends | `422 unprocessable_content` |
| Newly supplied `instanceId` cannot be resolved to a usable compute instance in the same scope | `422 unprocessable_content` |
| Resolved compute instance is on a different network | `422 unprocessable_content` |

The validation error for an unusable `instanceId` must be generic. It must not disclose whether the supplied ID was nonexistent, unreadable, or merely outside the caller's scope.

## Cross-Service Semantics

### Network Dependency

- `networkId` establishes a dependency edge from the load balancer to the consumed Region `Network`.
- If the referenced network enters `DELETING`, Region must initiate deletion of the dependent load balancer.
- The network is not blocked by the load balancer. Deleting the network triggers load balancer deletion instead of being rejected.

### Compute-Member Lifecycle

- Load balancer membership does not place a permanent blocking reference on member compute instances.
- Region subscribes to compute-instance lifecycle events through the platform event mechanism.
- On a compute-instance event, Region re-reads the compute instance through the Compute API and reconciles the affected load balancer.
- If a referenced instance's private IP changes, backend members are updated in place.
- If a referenced instance temporarily has no private IP, the load balancer remains present but not ready until the address is available or the member is removed.
- When a member instance enters `DELETING` or disappears, Region immediately withdraws the corresponding provider backend or member.
- Region does not mutate the public load-balancer `spec` during that withdrawal path. The stale member entry remains until the client removes or replaces it.
- A stale member entry that points at a deleted or deleting instance must not cause the backend to be recreated on later reconciles.

### Client Responsibilities

- Cluster API or other infrastructure clients should remove members as part of machine scale-down.
- Kubernetes CCM integrations should remove members when the corresponding node disappears.
- Clients must not rely on automatic public-spec cleanup.

### Non-Running Instance Behavior

- Stopped, errored, and rebooting instances are not auto-removed from the member list.
- With `healthCheck` enabled, provider health monitoring may pull those backends out of rotation.
- Without `healthCheck`, traffic may continue to be attempted to failed backends until the client updates the member list.
- This is a v1 limitation.

## Defaults and Derived Behavior

The initial public behavior is:

- `publicIP` is top-level on the load balancer spec.
- `allowedCidrs` is listener-scoped.
- `proxyProtocolV2` is v2-only, TCP-only, and defaults to `false`.
- `healthCheck` object presence means enabled.
- Health-check omission means disabled.
- `healthCheck: {}` means enabled with defaults.
- Health-check add, remove, and parameter updates are supported in place.
- Omitted `idleTimeoutSeconds` uses provider defaults.
- Zero effective backends, unresolved members, or only stale withdrawn members keep the load balancer present but not yet available.
- The implementation is IPv4-only.

Provider-specific protocol, health-monitor, timeout, and persistence mapping is defined in [OpenStack Load Balancers](../providers/openstack/load-balancers.md).
