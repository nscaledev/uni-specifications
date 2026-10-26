# Server Provider Create Gates

## Status

Draft.

## Background

Provider create gates define a generic pre-provider-create extension point for
Region `Server` resources. They are intended for service-owned pre-create work,
where another service must complete provider-specific preparation before Region
asks the infrastructure provider to create a server.

The model borrows one useful idea from Kubernetes Pod readiness gates:

- the desired gate list is part of the resource spec at creation time
- a gate is satisfied by a matching status entry becoming `True`
- a missing status entry is treated as not satisfied

Region differs from Pod readiness in the important part of the lifecycle. Pod
readiness gates decide whether an already-started Pod is service-ready. Server
provider-create gates block the provider create call itself. The Region
`Server` API object already exists; the gate applies to provider instance
creation.

## Goals

- Provide a generic pre-provider-create extension point for Region `Server`
  resources.
- Avoid a race between `Server` creation and the first server-controller
  reconcile.
- Keep Region as the only writer of Region-owned storage.
- Let service controllers discover pending work through the Region API.
- Keep provider-create gates separate from deletion blocking and provider
  cleanup.

## Non-Goals

- This design does not define the lifecycle of any service-owned external
  resources. The service that owns a gate remains responsible for preparing and
  cleaning up its own external state.
- This design does not define Region-level, Network-level, or Flavor-level gate
  defaulting. Gates are carried on the `Server` create request until there is a
  concrete reason to add a broader policy mechanism.
- This design does not let service callers mutate `Server.spec` after create.
- This design does not gate server deletion. Deletion remains driven by
  finalizers, references, and idempotent provider cleanup.

## API Model

### Stored Spec

`Server.spec.providerCreateGates` is the immutable list of gate condition types
that must be satisfied before the server controller calls the provider create
path.

Use the Kubernetes Pod readiness-gate shape rather than a bare string list:

```go
type ServerProviderCreateGate struct {
    // ConditionType is the status condition type that satisfies this gate.
    // It must use Kubernetes label-key format, for example:
    // example.unikorn-cloud.org/pre-create-ready.
    ConditionType string `json:"conditionType"`
}

type ServerSpec struct {
    // Existing fields omitted.

    // ProviderCreateGates must be present when the Server is created and is
    // immutable for the lifetime of the Server.
    ProviderCreateGates []ServerProviderCreateGate `json:"providerCreateGates,omitempty"`
}
```

The list must be map-like and keyed by `conditionType` in the storage schema so
a server cannot require the same gate twice.

### Stored Status

Arbitrary provider-create gate types should not be stored in a generic
conditions list if that list is reserved for service-owned lifecycle conditions.
Provider-create gate condition names need domain-qualified values such as
`example.unikorn-cloud.org/pre-create-ready`.

Store provider-create gate state in a server-specific status list with condition
semantics:

```go
type ServerProviderCreateGateStatus struct {
    // ConditionType matches ServerProviderCreateGate.ConditionType.
    ConditionType string `json:"conditionType"`
    // Status is True, False, or Unknown.
    Status corev1.ConditionStatus `json:"status"`
    // LastTransitionTime records when status last changed.
    LastTransitionTime metav1.Time `json:"lastTransitionTime"`
    // Actor is the mTLS service identity that last wrote this gate status.
    Actor string `json:"actor"`
    // Reason is a machine-readable reason.
    Reason string `json:"reason"`
    // Message is human-readable detail.
    Message string `json:"message"`
}

type ServerStatus struct {
    // Existing fields omitted.

    ProviderCreateGates []ServerProviderCreateGateStatus `json:"providerCreateGates,omitempty"`
}
```

Missing status for a configured gate is treated as `False` when computing
remaining gates. The stored status list is retained on the object for operators
and diagnostics; the normal service read API does not need to expose it.

### Server Read Status

The v2 Server API is internal to services. Its server read response must report
the remaining gate names so service controllers can decide whether they have
work to do.

Recommended response shape:

```yaml
status:
  remainingProviderCreateGates:
  - example.unikorn-cloud.org/pre-create-ready
```

`remainingProviderCreateGates` is derived from
`spec.providerCreateGates`: it contains each configured gate whose effective
status is not `True`. A controller can fetch a server and check whether its
condition type is in this list before doing external work.

The full `status.providerCreateGates` list remains in the stored object so
operators can inspect condition status, actor, reason, message, and transition
time.

## Initialization

Provider-create gates must be present on the `Server` at creation time. Adding
them after creation is not acceptable because the server controller could
reconcile the object and call the provider before the gates arrive.

The initial mechanism is explicit create-time input on the normal v2 Server
create API:

```yaml
spec:
  providerCreateGates:
  - conditionType: example.unikorn-cloud.org/pre-create-ready
```

Region persists that list into `Server.spec.providerCreateGates` before
creating the stored `Server` object. The field is immutable after creation.

The service that initiates Server creation owns the gate declaration. If a
different service owns the pre-create work, the creating service includes that
gate as part of its contract with the pre-create service. Region does not infer
or default gates from Region, Network, Flavor, or provider configuration.

This intentionally does not add a Region-level default such as
`Region.spec.serverProviderCreateGates`. If a broader policy is needed later,
it can be added with clearer requirements, for example only for specific
Networks, Flavors, Regions, or infrastructure references.

Callers do not add, remove, or edit gates after the `Server` exists. Service
controllers only report the status of gates already present on the `Server`.

Existing servers created before the field exists have an empty list and
continue to provision as they do today.

## Enforcement

The server controller checks provider-create gates immediately before it would
call the provider create path.

The controller may proceed only if every configured gate has effective status
`True`. Effective status is computed as:

- matching status entry with `status == True`: satisfied
- matching status entry with `status == False` or `Unknown`: remaining
- no matching status entry: remaining

If any gate remains, the controller yields and retries later. It must not call
the provider and must not mark the server as failed merely because a pre-create
service has not finished.

Region owns reset semantics. The server controller resets provider-create gate
status only when it intentionally returns the server to the pre-provider-create
state and will make a new provider create attempt from scratch. Ordinary
reconcile retries, watch replays, status-write conflicts, and waiting for an
existing provider operation must not reset gates.

When a reset is required, the controller sets every configured gate status to
`Unknown` with a Region-owned reason and message explaining that provider
create will be retried. It must not mutate `Server.spec.providerCreateGates`.
Service controllers discover the reset through
`status.remainingProviderCreateGates` and re-run their work.

Deletion does not check provider-create gates. Deprovisioning must continue to
run unconditionally and idempotently, consistent with normal controller
deletion semantics.

## Internal Service API

Other services and their controllers must act through the Region API, not by
writing Region-owned storage directly.

### Read

Service controllers read normal v2 server state:

```http
GET /api/v2/servers/{serverID}
```

The response includes:

- `status.regionId`
- `status.networkId`
- `status.infrastructureRef`
- `spec.flavorId`
- `status.remainingProviderCreateGates`

This is enough for a service controller to decide whether a server needs its
work and to gather the data needed to do that work.

### Satisfy Gate

Use a body field for `conditionType` rather than a path segment. Gate condition
types commonly contain `/`, and relying on path escaping for domain-qualified
condition names is fragile.

```http
POST /api/v2/servers/{serverID}/provider-create-gates
```

```json
{
  "conditionType": "example.unikorn-cloud.org/pre-create-ready",
  "reason": "Prepared",
  "message": "External pre-create state is ready"
}
```

Rules:

- the server must exist and be readable by the caller
- `conditionType` must already be present in `Server.spec.providerCreateGates`
- the endpoint marks the matching provider-create gate status `True`
- the endpoint records the supplied `reason` and `message` on the stored object
  for operator diagnostics
- the endpoint records the authenticated mTLS service identity as the gate
  status actor; callers do not provide this field
- the endpoint does not add or remove spec gates
- repeated writes with the same state are idempotent
- service callers cannot set a gate back to `False` or `Unknown`
- Region owns resets if the server controller needs pre-create work to run
  again

The internal API should have a dedicated RBAC scope, for example
`region:servers:v2/provider-create-gates`, so service accounts can be granted
this write without general server update permission.

No additional condition-type ownership check is required. Any service account
that is authorized for the provider-create-gate action may satisfy any
configured gate on a server it can read. Region must write a structured log for
each accepted gate action, including the server ID, condition type,
authenticated actor, reason, and whether the action changed the stored status.

### References

Deletion blocking is separate. If another service owns cleanup that must
complete before server deletion, Region should expose a Server references
subresource analogous to other reference subresources.

Provider-create gate status must not be used as a deletion-blocking mechanism.

## Example Use Case

A service that creates a gated `Server` includes:

```yaml
spec:
  providerCreateGates:
  - conditionType: example.unikorn-cloud.org/pre-create-ready
```

A pre-create service watches or polls Region server events through its normal
service integration. When it reads a server whose
`remainingProviderCreateGates` includes
`example.unikorn-cloud.org/pre-create-ready`, it:

1. reads the server's `infrastructureRef`, `networkId`, and `flavorId`
2. determines whether service-owned pre-create work is required
3. prepares external state when required
4. satisfies the provider-create gate through the Region API

If no work is required for a particular server, the service still satisfies the
gate. The gate means "the pre-provider-create service has made its decision",
not "the service necessarily changed external state".

## Implementation Requirements

- The v2 Server create API must accept `providerCreateGates` and persist them
  before the stored `Server` object is created.
- The stored `Server.spec.providerCreateGates` field must be immutable.
- The server controller must check the gates immediately before provider create.
- The server read API must expose `status.remainingProviderCreateGates`.
- The gate action endpoint must only satisfy a configured gate; it must not add
  gates, remove gates, or reset gates.
- The gate action endpoint must derive the actor from the authenticated mTLS
  service identity, record it in the stored gate status, and log accepted gate
  actions.
- The stored object should retain per-gate status, actor, reason, message, and
  transition time for operators.
