# OpenStack Load Balancers

This document defines the initial OpenStack provider mapping for the public Region v2 load balancer abstraction in [Region Load Balancers v2](../../region/load-balancers.md).

The initial provider assumption is Octavia with the Amphora driver. This is an implementation constraint for v1, not a permanent public API commitment.

## Changelog

| Version | Date | Notes |
|---|---|---|
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

Provider contract:

- `CreateLoadBalancer` is the idempotent reconcile entrypoint for both create and update.
- `DeleteLoadBalancer` fully tears down all provider resources, including floating IPs.
- `UpdateLoadBalancerState` refreshes derived status such as VIP and public IP.

## Reconciliation Semantics

The OpenStack provider implementation must satisfy the following behavior:

- Provider-side reconcile is idempotent across repeated calls.
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

| Octavia or local state | Region condition mapping |
|---|---|
| `PENDING_CREATE` or `PENDING_UPDATE` | `Available=False` with reason `ConditionReasonProvisioning` |
| `PENDING_DELETE` or local delete flow | `Available=False` with reason `ConditionReasonDeprovisioning` |
| `ACTIVE` with at least one effective backend and `status.vipAddress` populated | `Available=True` with reason `ConditionReasonProvisioned` |
| `ERROR` | `Available=False` with reason `ConditionReasonErrored` |
| Zero members or zero effective backends | `Available=False` with reason `ConditionReasonProvisioning` |

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
