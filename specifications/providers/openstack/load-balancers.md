# OpenStack Load Balancers

This document defines the initial OpenStack provider mapping for the public Region v2 load balancer abstraction in [Region Load Balancers v2](../../region/load-balancers.md).

The initial provider assumption is Octavia with the Amphora driver. This is an implementation constraint for v1, not a permanent public API commitment.

## Changelog

| Version | Date | Notes |
|---|---|---|
| v0.4 | 2026-04-12 | Compliance review: finalizer lifecycle, downstream error classification, deadlock detection, ErrYield queue semantics, reconciliation recovery, observability, expanded acceptance criteria |
| v0.3 | 2026-04-11 | Members use IP addresses directly; removed compute-instance lifecycle reconciliation; `Available=True` requires `status.vipAddress` populated |
| v0.2 | 2026-04-10 | Review update: listener-name identity, dependency-triggered deletion, stale-member withdrawal, and clarified status mapping |
| v0.1 | 2026-04-10 | Initial draft |

## Scope and Assumptions

The initial OpenStack implementation is:

- Octavia Amphora only
- Layer 4 only
- IPv4 only
- Public API remains provider-agnostic
- Hidden load-balancing algorithm fixed to `ROUND_ROBIN`

OVN compatibility is explicitly out of scope for this version, since OVN does not support VLANs currently.

## Resource Mapping

The public abstraction maps to Octavia and Neutron as follows:

| Region abstraction | OpenStack resource or behavior |
|---|---|
| One Region load balancer | One Octavia load balancer |
| One Region listener | One Octavia listener |
| One Region pool | One Octavia pool |
| One Region member | One Octavia member using the specified member address and configured member port |
| `publicIP=true` | One Neutron floating IP attached to the VIP port |
| Listener `allowedCidrs` | Octavia listener allowed-CIDRs feature (`allowed_cidrs` API field) |
| Hidden public algorithm | Octavia `ROUND_ROBIN` |

`networkId` is the tenant network used for both the VIP subnet selection and backend member reachability.

## Protocol and Health-Monitor Mapping

| Public listener settings | Octavia listener protocol | Octavia pool protocol | Octavia health monitor type |
|---|---|---|---|
| `tcp` with `proxyProtocolV2=false` | `TCP` | `TCP` | `TCP` |
| `tcp` with `proxyProtocolV2=true` | `TCP` | `PROXYV2` | `TCP` |
| `udp` | `UDP` | `UDP` | `UDP-CONNECT` |

`proxyProtocolV2=true` means the backend hop uses PROXY protocol version 2. The client-facing listener protocol remains plain TCP.

## Timeout Mapping

`idleTimeoutSeconds` maps only to the Amphora data-path idle timeouts:

- `timeout_client_data = idleTimeoutSeconds * 1000`
- `timeout_member_data = idleTimeoutSeconds * 1000`

The following remain at provider defaults:

- `timeout_member_connect`
- `timeout_tcp_inspect`

## Controller and Persistence Model

The implementation adds:

- A new user-facing Region CRD: `LoadBalancer`
- A new OpenStack persistence CRD: `OpenstackLoadBalancer`

`OpenstackLoadBalancer` must persist at minimum:

| Scope | Required fields |
|---|---|
| Load balancer | `loadBalancerID`, `vipPortID`, `floatingIPID?` |
| Listener | `name`, `listenerID`, `poolID`, `healthMonitorID?`, last applied `allowedCidrs`, last applied `idleTimeoutSeconds?`, last applied `proxyProtocolV2` |
| Member | `listenerName`, `address`, `memberID`, `port` |

Listener reconciliation is keyed by the public `listener.name`, not by provider-generated IDs alone. Member persistence must be scoped per listener or pool because the same address may appear in multiple pools with different ports.

Extend the Region provider interfaces with:

- `CreateLoadBalancer`
- `DeleteLoadBalancer`
- `UpdateLoadBalancerState`

These are in-process Go interfaces called directly by the controller within the Region service. They do not cross an API boundary and require no authentication classification or OpenAPI annotation.

Provider contract:

- `CreateLoadBalancer` is the idempotent reconcile entrypoint for both create and update.
- `DeleteLoadBalancer` fully tears down all provider resources, including floating IPs.
- `UpdateLoadBalancerState` refreshes derived status such as VIP and public IP.

## Reconciliation Semantics

The OpenStack provider implementation must satisfy the following behavior:

- Provider-side reconcile is idempotent across repeated calls. If the controller crashes mid-reconcile, the next reconcile pass reads persisted IDs from `OpenstackLoadBalancer` and resumes from the last successful state. Partial provider state is handled by the subsequent reconcile pass without manual intervention.
- Listener reconciliation is keyed by stable public `listener.name`.
- A listener rename is handled as delete-and-create of the provider listener and pool subtree.
- Listener CIDRs are updated in place.
- Listener idle timeout is updated in place.
- Health monitor create, delete, and parameter updates are reconciled in place.
- Member sets and member ports are updated in place.
- `proxyProtocolV2` changes for TCP listeners are reconciled without changing the public resource contract.
- Toggling `publicIP` attaches or detaches the Neutron floating IP from the VIP port.

## Public Status Exposure

The Region API exposes only provider-agnostic status fields:

- `status.regionId`
- `status.networkId`
- `status.vipAddress`
- `status.publicIP`
- Standard `status.conditions`

The following are not exposed in v1:

- Octavia IDs
- Per-listener operating status
- Per-member health
- Statistics
- The status tree
- The selected algorithm
- The driver name

## Condition Mapping

For status derivation, an effective backend is a member that has been successfully provisioned as an Octavia pool member.

Every reconcile pass, successful or not, must update the `Available` condition before returning. This is unconditional per [SPECIFICATION.md §8.4](../../SPECIFICATION.md).

| Octavia or local state | Region condition mapping | Reconcile return | Queue behavior |
|---|---|---|---|
| `PENDING_CREATE` or `PENDING_UPDATE` | `Available=False` with reason `ConditionReasonProvisioning` | `ErrYield` | Fixed interval requeue |
| `PENDING_DELETE` or local delete flow | `Available=False` with reason `ConditionReasonDeprovisioning` | `ErrYield` | Fixed interval requeue |
| `ACTIVE` with at least one effective backend and `status.vipAddress` populated | `Available=True` with reason `ConditionReasonProvisioned` | `nil` | Removed from queue |
| `ERROR` | `Available=False` with reason `ConditionReasonErrored` | error | Exponential backoff |
| Zero members or zero effective backends | `Available=False` with reason `ConditionReasonProvisioning` | `ErrYield` | Fixed interval requeue |

The zero-members case uses `ConditionReasonProvisioning` because the load balancer is not yet fully operational, even though the Octavia LB itself may be `ACTIVE` with a VIP allocated. This is the best-fit standard reason within the platform's condition vocabulary. Callers should not interpret `ConditionReasonProvisioning` as necessarily meaning infrastructure provisioning is in progress.

If the status write itself fails (e.g., resource version conflict), the controller requeues with a fixed timeout and does not return an error (status write failure is transient per [SPECIFICATION.md §8.4](../../SPECIFICATION.md)).

## Finalizer Lifecycle

The controller adds its finalizer to the `LoadBalancer` resource before performing any provisioning work per [SPECIFICATION.md §8.5](../../SPECIFICATION.md).

On deletion:

1. The controller detects the deletion timestamp on the `LoadBalancer` resource.
2. While inbound references or owned children exist, the controller retries silently (`ErrYield`).
3. The controller calls `DeleteLoadBalancer`, which fully tears down all Octavia resources (listeners, pools, members, health monitors, the load balancer itself), detaches and deletes any Neutron floating IP, and clears the `OpenstackLoadBalancer` persistence state.
4. The controller releases any outbound references (none in the current design, since the Network edge is a dependency, not a consumption).
5. Only after all provider resources and references are cleaned up does the controller remove the finalizer.

## Downstream Error Handling

When the controller receives error responses from Octavia or Neutron API calls, it classifies them per [SPECIFICATION.md §8.7](../../SPECIFICATION.md):

| Octavia/Neutron response | Treatment |
|---|---|
| 5xx or service unavailable | Transient. Return `ErrYield`, retry at fixed interval. |
| 409 conflict (e.g., Octavia `IMMUTABLE`) | Transient. Re-read provider state and retry at fixed interval. |
| 404 on a resource the controller expects to exist | Genuine error. Surface as `ConditionReasonErrored`. Data inconsistency that will not resolve on retry. |
| 403 | Genuine error. Surface as `ConditionReasonErrored`. Permission failure will not resolve without intervention. |
| 401 | Transient for a bounded number of retries (credential refresh). If persistent, surface as `ConditionReasonErrored`. |

The controller must not poll Octavia to check whether dependent resources are ready. Dependency readiness is inferred from local status conditions via status propagation upward per [SPECIFICATION.md §8.7](../../SPECIFICATION.md).

## Deadlock Detection

On every reconcile pass of a load balancer in `DELETING` state, the controller checks the age of the deletion timestamp. If the timestamp has been set for longer than 10 minutes and inbound references are still present, the controller emits a structured zap error log entry per [SPECIFICATION.md §8.8](../../SPECIFICATION.md).

The log entry must include as discrete structured zap fields (indexable in Loki): resource type (`LoadBalancer`), resource ID, deletion timestamp, elapsed duration, and the full set of blocking reference strings.

## Observability

The controller follows the platform observability requirements in [SPECIFICATION.md §9](../../SPECIFICATION.md):

- **Structured logging**: every reconcile pass, state transition (Octavia state changes, floating IP attach/detach), reference operation, and requeue decision is logged with resource type and resource ID as structured zap fields. Verbose controller logging per [SPECIFICATION.md §9.1](../../SPECIFICATION.md).
- **Distributed tracing**: a span is created for every controller reconcile pass. Outbound calls to Octavia and Neutron propagate trace context.
- **Metrics**: the controller emits reconcile duration (histogram), reconcile outcome (counter by success/transient/error), work queue depth (gauge), and reference operation count (counter by add/remove and outcome). Metric names follow Prometheus conventions: `snake_case`, service-prefixed, with unit suffixes.

## Explicitly Out of Scope

This provider mapping does not include:

- OVN compatibility
- HTTP or HTTPS listeners
- Terminated TLS
- L7 routing or policies
- Weighted members
- Backup members
- Session persistence
- Alternate monitor IP
- Alternate monitor port
- HTTP, HTTPS, TLS-HELLO, PING, or SCTP health checks
- Advanced Amphora timeout knobs beyond `idleTimeoutSeconds`
- IPv6 VIPs
- IPv6 members
- Mixed IPv4 and IPv6 members
- Additional VIPs
- Listener connection limits

## Acceptance Criteria

Implementation handoff must include tests that cover:

- Create a private TCP load balancer with one or more IP-address members.
- Create a private UDP load balancer with one or more IP-address members.
- Create a public load balancer with `publicIP=true`.
- Create a load balancer with `members: []` and verify VIP allocation succeeds while `Available` stays false until a backend becomes effective.
- Create one load balancer with multiple listeners and different `allowedCidrs` per listener.
- Create a TCP listener with `proxyProtocolV2=true`.
- Reject `proxyProtocolV2` on UDP listeners.
- Create a load balancer with no `healthCheck`.
- Create a load balancer with `healthCheck: {}`.
- Create a TCP listener with explicit health-check values.
- Create a UDP listener with explicit health-check values.
- Create a TCP listener with `idleTimeoutSeconds`.
- Reject `idleTimeoutSeconds` on UDP listeners.
- Update listener CIDRs in place.
- Verify `allowedCidrs: []` and omitted `allowedCidrs` both mean unrestricted after update.
- Update listener idle timeout in place.
- Update a listener by name and verify rename is treated as delete-and-create.
- Update member sets and member ports in place.
- Create, remove, and update a health monitor in place.
- Update `proxyProtocolV2` in place for TCP listeners.
- Update `publicIP` in place.
- Reject duplicate listener names.
- Reject duplicate `(protocol, port)` listeners.
- Reject invalid CIDRs.
- Reject a member with an invalid IPv4 address.
- Reject duplicate `(address, port)` members within one pool.
- Allow the same address in different pools.
- Allow the same address with different ports in one pool.
- Verify network deletion triggers load balancer deletion and is not blocked by the load balancer.
- Verify member address and port updates are reconciled to Octavia in place.
- Verify Amphora Octavia resources, floating IPs, and references are fully cleaned up on delete.
- Verify `PUT` returns `409 conflict` when the resource version has advanced (conflict detection).
- Verify `DELETE` on an already-deleting resource returns `202` without re-triggering deletion (idempotent delete).
- Reject load balancer creation when the network is in a different organisational scope (co-location validation returns `422`).
- Verify the controller finalizer prevents premature garbage collection during deprovisioning.
- Verify RBAC enforcement for the `region:loadbalancers:v2` scope on all endpoints.
- Verify behavior when `region.features.loadBalancers` is `false` for the target region.
