# Region Load Balancers v2

This document defines the public `uni-region` v2 load balancer API. It is the normative public abstraction for Layer 4 load balancing. Provider-specific mapping for the initial OpenStack Octavia Amphora implementation is documented in [OpenStack Load Balancers](../providers/openstack/load-balancers.md).

The standard Region v2 resource envelope and the platform-wide rules in [SPECIFICATION.md](../../SPECIFICATION.md) still apply. This document defines the load-balancer-specific API surface, schema, validation rules, and cross-service semantics.

## Changelog

| Version | Date | Notes |
|---|---|---|
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
- Listener connection limits
- Raw IP members

## API Surface

Add the following Region v2 endpoints:

- `GET /api/v2/loadbalancers`
- `POST /api/v2/loadbalancers`
- `GET /api/v2/loadbalancers/{loadBalancerID}`
- `PUT /api/v2/loadbalancers/{loadBalancerID}`
- `DELETE /api/v2/loadbalancers/{loadBalancerID}`

All write operations are asynchronous and return `202 Accepted`. Callers must read `status.conditions` to determine whether provisioning has completed.

List filters follow existing Region v2 conventions:

- `organizationID`
- `projectID`
- `regionID`
- `networkID`
- `tag`

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
- The public status surface is provider-agnostic. Provider IDs, per-listener operating status, per-member health, statistics, the algorithm, the status tree, and driver names are not exposed in v1.

## Listener Schema

Each listener is represented by `LoadBalancerListenerV2`:

| Field | Required | Description |
|---|---|---|
| `name` | yes | Listener name, unique within the load balancer. |
| `protocol` | yes | `tcp` or `udp`. |
| `port` | yes | Frontend port number. |
| `allowedCidrs` | no | IPv4 CIDRs allowed to reach this listener. |
| `idleTimeoutSeconds` | no | TCP-only idle timeout override. |
| `pool` | yes | Exactly one backend pool for this listener. |

Listener rules:

- `name` must be unique within the load balancer.
- `(protocol, port)` must be unique within the load balancer.
- `allowedCidrs` uses OR semantics.
- Omitted or empty `allowedCidrs` means unrestricted ingress for that listener.
- Every `allowedCidrs` entry must be a valid IPv4 CIDR.
- `idleTimeoutSeconds` is valid only for `tcp`.
- Omitted `idleTimeoutSeconds` means provider default.

## Pool Schema

Each listener owns exactly one `LoadBalancerPoolV2`:

| Field | Required | Description |
|---|---|---|
| `proxyProtocolV2` | no | TCP-only backend PROXY protocol version 2 toggle. |
| `members` | yes | Backend members. |
| `healthCheck` | no | Optional transport-level health check. |

Pool rules:

- Exactly one pool exists per listener.
- `proxyProtocolV2` is valid only for `tcp` listeners.
- `proxyProtocolV2=true` means the backend connection is wrapped in PROXY protocol version 2.
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
- `port` is required.
- Duplicate `instanceId` values in one pool are rejected.
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
- `timeoutSeconds` must be less than `intervalSeconds` when a health check is present.
- Health checks always target the member's configured backend port using transport semantics derived from the listener protocol and pool settings.

## Validation and Error Semantics

The request body is first validated against the OpenAPI schema. Schema or field-format failures should return `400 invalid_request`. Cross-field or cross-resource semantic failures should return `422 unprocessable_content`.

Create and update validation must enforce all of the following:

- `organizationId`, `projectId`, `regionId`, and `networkId` remain immutable after create.
- Every `instanceId` resolves through the Compute v2 API.
- Every referenced compute instance belongs to the same organization, project, region, and network as the load balancer.
- Referenced compute instances may exist before private IP assignment. In that case the load balancer remains provisioning until the address resolves.
- `proxyProtocolV2=true` is rejected for `udp`.
- `idleTimeoutSeconds` is rejected for `udp`.
- `timeoutSeconds` must be less than `intervalSeconds` when a health check is present.
- Duplicate listener names are rejected.
- Duplicate `(protocol, port)` listeners are rejected.
- Duplicate `instanceId` members are rejected.
- Duplicate resolved `(privateIP, port)` backends are rejected.

Preferred response behavior:

| Condition | Preferred response |
|---|---|
| Invalid IPv4 CIDR syntax in `allowedCidrs` | `400 invalid_request` |
| Unsupported field combinations such as `proxyProtocolV2=true` on `udp` or `idleTimeoutSeconds` on `udp` | `422 unprocessable_content` |
| `timeoutSeconds >= intervalSeconds` | `422 unprocessable_content` |
| Duplicate listener names, duplicate `(protocol, port)` listeners, duplicate `instanceId` members, or duplicate resolved `(privateIP, port)` backends | `422 unprocessable_content` |
| Supplied `instanceId` cannot be resolved to a usable compute instance in the same scope | `422 unprocessable_content` |
| Resolved compute instance is on a different network | `422 unprocessable_content` |

The validation error for an unusable `instanceId` must be generic. It must not disclose whether the supplied ID was nonexistent, unreadable, or merely outside the caller's scope.

## Cross-Service Semantics

The Region load balancer controller maintains permanent references to:

- The consumed Region `Network`
- Every referenced `uni-compute` compute instance

The resource graph semantics are:

- Network deletion is blocked while a load balancer exists.
- Referenced compute-instance deletion is blocked while it remains a member.
- References are reconciled on every pass so removed members release references.
- Region subscribes to compute-instance lifecycle events through the platform event mechanism.
- On a compute-instance event, Region re-reads the compute instance through the Compute API and reconciles the affected load balancer.
- If a referenced instance's private IP changes, backend members are updated in place.
- If a referenced instance temporarily has no private IP, the load balancer remains present but not ready until the address is available or the member is removed.

## Defaults and Derived Behavior

The initial public behavior is:

- `publicIP` is top-level on the load balancer spec.
- `allowedCidrs` is listener-scoped.
- `proxyProtocolV2` is v2-only and TCP-only.
- `healthCheck` object presence means enabled.
- Health-check omission means disabled.
- `healthCheck: {}` means enabled with defaults.
- Omitted `idleTimeoutSeconds` uses provider defaults.
- The implementation is IPv4-only.

Provider-specific protocol, health-monitor, timeout, and persistence mapping is defined in [OpenStack Load Balancers](../providers/openstack/load-balancers.md).
