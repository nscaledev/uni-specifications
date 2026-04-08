# Minimal Reservation Service — Design

## Overview

The reservation service pre-allocates bare metal hosts for a VPC before any server is booted. It is OpenStack-specific: it talks to Nova and Ironic, and its internal CRDs model OpenStack concepts directly.

**The problem it solves.** Bare metal host selection by Nova placement is opportunistic — at `nova boot` time, placement picks whatever host satisfies the flavour constraints. For workloads that need pre-boot infrastructure programming (IB partition assignment, network programming, topology-aware placement), the host must be known and programmed before the boot call. The reservation service pins the host early and holds it, allowing dependent services to complete their programming against a stable host identity before any instance runs.

**What the reservation service does.** The operative concept in the minimal design is the *Placement*. A caller creates a Placement specifying a region, a flavour, a VPC, and a set of readiness gates. The controller selects available hosts of the right flavour, creates a Nova host aggregate containing those hosts, and records the aggregate ID and selected host list in status. External services (e.g. ib-manager) receive a Placement creation event, perform their per-host programming, and signal readiness by setting a named condition on the Placement via the reservation service REST API. Once all gates are satisfied and the Placement is marked Ready, callers can create Servers: the region service reads the aggregate ID from the Placement and passes it to Nova as a scheduler hint, constraining placement to the pre-programmed hosts.

**Scope.** This design covers the minimal case: one host per Placement. Multi-host Placements with topology constraints, and Reservations as a capacity management layer above Placements, are planned expansions but not designed here.

---

## Design

### CRD: `OpenStackPlacement`

The `OpenStackPlacement` CRD is internal to the reservation service. Its schema uses OpenStack terminology directly — there is no provider abstraction.

```go
type OpenStackPlacementSpec struct {
    // Region in which to allocate a host
    RegionID string
    // VPC (Network) this Placement is associated with.
    // External services (e.g. ib-manager) use this to determine which
    // partition or network resource to program for the placed hosts.
    NetworkID string
    // Flavour determining which class of host to select
    FlavorID string
    // Condition types that must be True before this Placement is usable.
    // Copied verbatim from Region.Spec.ReadinessGates at creation time.
    ReadinessGates []string
}

type OpenStackPlacementStatus struct {
    // Nova host aggregate UUID created for this Placement.
    // Also serves as the disjointness signal: hosts in this aggregate
    // are ineligible for selection by any other Placement.
    AggregateID string
    // Ironic node UUIDs of the hosts assigned to this Placement.
    // Set by the controller after host selection; immutable thereafter.
    // Read by external services (e.g. ib-manager) to discover which
    // hosts to program.
    HostIDs []string
    // Standard Kubernetes conditions.
    // Includes the controller-owned Ready condition and any gate conditions
    // set by external services.
    Conditions []metav1.Condition
}
```

**Conditions:**
- `Allocated` — set True by the controller once hosts have been selected, added to the aggregate, and recorded in `Status.HostIDs`. This is the controller's own signal that the Placement has been populated.
- Gate conditions (e.g. `ib.unikorn-cloud.org/partition-ready`) — set by external services via the REST API, as named in `Spec.ReadinessGates`.
- `Ready` — set True by the controller only when `Allocated` is True and every condition named in `Spec.ReadinessGates` is also True.

Consumers (e.g. the Server provisioner) wait only for `Ready == True`. They do not inspect individual gate conditions — that aggregation is the controller's responsibility.

**Immutability.** `Status.HostIDs` does not change after it is first set. This is required so that the readiness gate fires exactly once and the programmed state of external services (e.g. the UFM partition) remains stable.

### Controller

On creation (DeletionTimestamp nil):

```
1. Select available hosts
       See Host Selection below

2. Create Nova host aggregate
       POST /compute/v2/os-aggregates
       Add the selected hosts to the aggregate

3. Record in status
       Status.AggregateID = aggregate UUID
       Status.HostIDs = [ironic node UUIDs]
       Set Allocated condition True

4. Recompute Ready
       If Allocated == True and all Spec.ReadinessGates conditions are True:
           Set Ready condition True
```

On deletion (DeletionTimestamp non-nil):

```
1. If GetResourceReferences() is non-empty: requeue, do not proceed
   (inbound references signal that a dependent service has not yet
   reversed its programming)

2. Delete Nova host aggregate
       DELETE /compute/v2/os-aggregates/{id}

3. Remove controller finalizer — Placement is deleted
```

The controller adds its own finalizer on creation to ensure the deletion path runs before the object is removed from etcd.

### Host Selection

The controller queries the OpenStack Placement API to find available hosts, then filters for hosts not already committed to another Placement.

**Step 1 — Resolve the resource class from the flavour.**
The flavour's extra specs contain `resources:CUSTOM_BAREMETAL_LARGE=1` (or equivalent). The controller queries Nova for the flavour and reads the `resources:CUSTOM_*` key to obtain the resource class name.

**Step 2 — Query Placement for available hosts.**

```
GET /placement/resource_providers?resources=CUSTOM_BAREMETAL_LARGE:1
```

This returns only resource providers with free capacity. This is the correct source of truth because a node mid-deploy is already marked consumed in Placement even though its Ironic provision state is still `deploying`.

**Step 3 — Filter by aggregate membership.**
Each candidate is checked against Nova: any host already a member of an existing Placement aggregate is excluded. This is the disjointness mechanism — it ensures a host cannot be assigned to more than one Placement without requiring cross-CRD scanning.

**Step 4 — Cross-check against Ironic.**
Placement has a ~60-second sync lag from Ironic. To guard against stale data, each remaining candidate is checked directly against Ironic:
- `provision_state == available`
- `maintenance == false`

A host passing all checks is selected.

If no host is available, the controller sets a `HostUnavailable` condition and requeues. Host selection is best-effort with no distributed lock — in the event of a race between two concurrent Placements selecting the same host, the aggregate-membership check reduces but does not eliminate the window. The minimal service accepts this: single-host Placements with human-driven scheduling make races rare in practice.

### Resource References

External services that perform per-host programming register a deletion block on the Placement while their programming is active. This prevents the aggregate from being torn down before the programming is reversed.

Per the platform spec (section 5.3), references cross service boundaries via the owning service's reference REST API — external services never write another service's finalizers directly. The reservation service exposes `PUT` and `DELETE` reference endpoints; internally it translates these into finalizer operations on its own CRD. The reference string format follows the canonical `{resource}.{group}/{uuid}` scheme produced by `GenerateResourceReference`.

**Lifecycle:**

```
ib-manager programs IB partition
    → PUT .../placements/{id}/references/{ref}
    → reservation service adds finalizer to Placement CRD

Placement deleted (DeletionTimestamp set)
    → controller sees inbound reference via GetResourceReferences(), requeues

ib-manager receives deletion event, reverses UFM programming
    → DELETE .../placements/{id}/references/{ref}
    → reservation service removes finalizer from Placement CRD

No references remain
    → controller proceeds: deletes aggregate, removes its own finalizer
```

### Deletion Protocol

Full end-to-end deletion sequence:

```
User deletes Placement
    │ DeletionTimestamp set
    ▼
Event bus fires → external services receive deletion notification
    ib-manager:
        Removes host GUIDs from UFM partition
        DELETE .../placements/{id}/references/{ref}

Placement controller (requeued):
    GetResourceReferences() non-empty?
        Yes → requeue
        No  → DELETE Nova aggregate
              Remove controller finalizer
              Placement removed from etcd
```

---

## REST API

All endpoints are called by external services (region service, ib-manager). The reservation service writes to its own CRDs in response; callers never touch CRDs directly.

| Method | Path | Caller | Purpose |
|---|---|---|---|
| `POST` | `/placements` | Region service | Create a Placement |
| `GET` | `/placements/{id}` | Region service, ib-manager | Read full Placement state (host list, aggregate ID, conditions) |
| `DELETE` | `/placements/{id}` | Region service | Delete a Placement |
| `PUT` | `/placements/{id}/conditions/{type}` | ib-manager | Set a named condition (e.g. `ib.unikorn-cloud.org/partition-ready: True`) |
| `PUT` | `/placements/{id}/references/{name}` | ib-manager | Register a deletion block (idempotent) |
| `DELETE` | `/placements/{id}/references/{name}` | ib-manager | Release a deletion block (idempotent) |

All calls use mTLS; the caller's service account certificate is its identity.

---

## Integration with the Region Service

### Creating a Placement

The region service creates a Placement when a Network is provisioned in an IB-capable region (or more generally, whenever the Region's `Spec.ReadinessGates` is non-empty). The Placement is created with:
- `RegionID` from the Network's region
- `NetworkID` from the Network being provisioned
- `FlavorID` from the intended Server flavour
- `ReadinessGates` copied verbatim from `Region.Spec.ReadinessGates`

### Booting a Server

```
Region service creates Server

    1. Fetch Placement via reservation service REST API
    2. Check Ready == True
           If False or absent: Server stays pending, requeue

    3. Ready == True:
           Read Status.AggregateID from Placement
           POST /compute/v2/servers with scheduler hint:
               os:scheduler_hints:
                 aggregate_instance_extra_specs:
                   unikorn-placement: {placement-id}
           (Aggregate metadata key set by reservation service at aggregate creation time)
```

Nova placement selects the single host within the aggregate. Because only one host is in the aggregate and all pre-boot programming is complete, placement is deterministic.

### Placement-to-Server relationship

In the minimal service, one Placement is consumed by at most one Server. The region service is responsible for enforcing this: once a Server has been booted against a Placement, the Placement is considered consumed and should not be reused.

---

## Integration with ib-manager

ib-manager subscribes to Placement lifecycle events via the Kubernetes event bus (informer on the `OpenStackPlacement` CRD). The envelope carries `ResourceID` and `DeletionTimestamp`.

**On creation (DeletionTimestamp nil):**
1. `GET /placements/{id}` — fetch full state including `Spec.NetworkID` and `Status.HostIDs`
2. Fetch flavour via region service REST API → `FlavorInfiniBandSpec.PortCount`
3. If `PortCount == 0`: set `ib.unikorn-cloud.org/partition-ready` True — done
4. Look up the `IBPartition` for `Spec.NetworkID`
5. For each host in `Status.HostIDs`, query Ironic for IB port GUIDs
6. Program UFM partition (add GUIDs to the `IBPartition`)
7. `PUT /placements/{id}/references/{ref}` — register deletion block
8. `PUT /placements/{id}/conditions/ib.unikorn-cloud.org/partition-ready` True
   → reservation service controller requeues, recomputes Ready

**On deletion (DeletionTimestamp non-nil):**
1. `GET /placements/{id}` — fetch current state
2. Remove host GUIDs from UFM partition
3. `DELETE /placements/{id}/references/{ref}` — release deletion block

---

## Event Bus

The reservation service publishes Placement lifecycle events to the Kubernetes event bus using the same envelope pattern as the region service: `ResourceID` and `DeletionTimestamp`. Subscribers (ib-manager) wire up a Kubernetes informer on the `OpenStackPlacement` CRD type.

`DeletionTimestamp == nil` → creation or update event.
`DeletionTimestamp` non-nil → deletion in progress; subscriber should reverse its programming.

---

## Future Expansion

The minimal design is deliberately constrained to one host per Placement and no topology constraints. The intended expansion path:

**Multi-host Placements.** Add `Spec.Count int` (defaulting to 1) to support multi-host Placements. `Status.HostIDs` is already a slice — no schema change needed. Host selection generalises from "pick one" to "pick N satisfying the topology constraint."

**Topology constraints.** Add a `Spec.Topology` field describing the placement policy: rack affinity (contiguous racks, hard-fail if unavailable), rack spread (anti-affinity with max-skew per rack, soft or hard), and similar. Topology is evaluated at host selection time within the Placement controller. The aggregate membership disjointness check runs first; the topology filter runs over the remaining candidates. This covers use cases P1 (rack-contiguous production) and P2 (cross-rack dev spread) from the use-case set.

**Reservations as a capacity management layer.** An `OpenStackReservation` CRD can be added above Placements to provide advance capacity booking and multi-Placement capacity management. A Reservation claims N units of a given flavour (possibly time-bound) and the Placement draws from that reserved pool rather than the open pool. This enables use cases R3 (future-dated bookings) and R4 (ranked fallback) and allows a policy decision about whether unreserved Placements are permitted. In the minimal design this layer is absent and all capacity is first-come first-served.

---

## Decision Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-04-01 | `Spec.NetworkID` carried on Placement | External services (e.g. ib-manager) need the VPC association to know which partition or network resource to program. Carrying it in Spec avoids a separate lookup and makes the Placement self-describing. |
| 2026-04-01 | Deletion blocking via reference REST API, not direct finalizer writes | Per platform spec section 5.3, external services never write another service's finalizers directly. The reservation service exposes canonical `PUT`/`DELETE` reference endpoints and manages finalizers internally. |
| 2026-04-01 | CRD named `OpenStackPlacement` | The CRD is OpenStack-specific (Nova aggregates, Ironic node UUIDs). Making the provider explicit in the type name is consistent with the design's stated principle of no provider abstraction, and avoids naming conflicts if other provider types are added later. |
| 2026-04-01 | Host selection via Placement API, cross-checked against Ironic | `GET /placement/resource_providers?resources=<class>:1` gives a more accurate availability view than querying Ironic directly — it reflects Nova allocations for nodes mid-deploy. Resource class is resolved from the flavour's `resources:CUSTOM_*` extra spec. Ironic is still checked to guard against Placement's ~60s sync lag. |
| 2026-04-08 | Disjointness enforced via Nova aggregate membership | A host already in any Placement aggregate is excluded from selection by a new Placement. This uses Nova's own state as the source of truth rather than scanning CRDs, and provides the same guarantee with less coordination. |
| 2026-04-08 | Topology constraints are a Placement concern, not a Reservation concern | Topology is a placement policy — where to put hosts — not a capacity policy. Rack affinity and spread are placement use cases. The Reservation layer (when added) expresses count and time bounds; the Placement expresses topology. |
| 2026-04-08 | Minimal design uses Placements only; Reservations deferred | IB partitioning requires only a stable host list, a VPC binding, and readiness gates — all of which are Placement properties. The Reservation layer adds capacity management and advance booking, which are not needed for the minimal IB case. Deferring it removes one CRD and one controller from the minimal implementation. |

---

## Migration

The current system boots servers directly via Nova with no pre-placement, no Placements, and no readiness gates. The migration must be non-breaking for existing regions and existing servers.

### Existing regions — no change required

`Region.Spec.ReadinessGates` is a new optional field that defaults to empty. An empty gate list means no gates are required before a Placement is used as a scheduler hint — and with no Placement in the boot path at all, existing regions behave exactly as before. No operator action is needed for regions that do not have an IB fabric.

### New IB-capable regions — opt-in by operator

A region with an IB fabric is configured with `ib.unikorn-cloud.org/partition-ready` in `Region.Spec.ReadinessGates` at deploy time. Only these regions use Placements and gate-based pre-boot coordination. Enabling ib-manager for a region is a single operator change to the Region definition; no flavour updates or server changes are needed.

### Existing servers — no backfill

Servers already running have no associated Placement. They do not need one: IB partitions for running servers were never programmed (there was no ib-manager), and retrofitting a Placement to a running server would not change the hardware state. The Server provisioner must handle both cases:

- **No Placement reference on the Server:** use the existing direct Nova placement path (current behaviour, unchanged).
- **Placement reference present:** check all gates, then boot with the aggregate scheduler hint.

This conditional is the only place in the codebase where the old and new paths diverge. Existing servers continue to work without modification.

### Schema changes are additive

All CRD changes introduced by this design are additive optional fields:

| Resource | New field | Default | Impact on existing resources |
|---|---|---|---|
| `Region` | `Spec.ReadinessGates []string` | `[]` (empty) | None — existing Regions have no gates |
| `OpenStackPlacement` | Entire new CRD | — | None — no existing Placements |
| `Flavor` | `Spec.InfiniBand.PortCount int` | `0` | None — existing Flavours have zero IB ports |

No data migration is required. Existing resources are valid under the new schema without modification.

### Rollout order

1. Deploy the updated region service with `Region.Spec.ReadinessGates` support and the conditional Server provisioner path.
2. Deploy the reservation service (or activate it within `uni-region` depending on the placement decision).
3. Deploy ib-manager.
4. For each IB-capable region, add `ib.unikorn-cloud.org/partition-ready` to `Region.Spec.ReadinessGates`.

Steps 1–3 have no user-visible effect. Step 4 activates the new path for that region only.

---

## Service Placement

Two deployment options exist. The choice has not yet been made.

### Option A: Embedded in the Region Service

The Placement controller and handler live inside `uni-region`. The `OpenStackPlacement` CRD belongs to the region service API group.

**Advantages:**
- Host selection can use internal region service state directly — flavour-to-resource-class mapping, Ironic client, and existing host inventory are all in-process. No new endpoints needed.
- One less service to deploy and operate.
- The Placement is a natural extension of region concepts (flavours, hosts, aggregates).

**Disadvantages:**
- The region service grows larger. Placement lifecycle (host selection, aggregate management, gate coordination) is a distinct concern from network and identity management.
- All consumers (ib-manager, future gate services) already call the region service; adding placement endpoints there conflates two separate responsibilities.

### Option B: Separate Service (`uni-reservation`)

The reservation service is a standalone binary with its own CRDs, API, and deployment.

**Advantages:**
- Clean separation of concerns. The reservation service owns host pre-allocation; the region service owns networks and identities.
- Independently deployable and scalable.
- The service boundary makes the gate protocol explicit — ib-manager calls `uni-reservation`, not `uni-region`.

**Disadvantages:**
- Host selection requires knowing which Ironic nodes are available for a given flavour. This state currently lives in the region service. A separate service would need either:
  - A new region service endpoint — "list available hosts for flavour X" — which does not exist today, or
  - Its own Ironic credentials and flavour-to-resource-class mapping logic, duplicating region service internals.
- Additional operational complexity: new deployment, new mTLS certificate, new RBAC role.

---

## Open Questions

1. **Host selection race:** The minimal service accepts a best-effort selection with no distributed lock. The aggregate-membership filter reduces the race window but does not eliminate it: two concurrent Placements can both pass the filter before either has created its aggregate. Is this acceptable for the target deployment?

2. **Aggregate scheduler hint mechanism:** The design uses `aggregate_instance_extra_specs` with a `unikorn-placement` metadata key on the aggregate, requiring `AggregateInstanceExtraSpecsFilter` in Nova's scheduler filter list. This needs to be confirmed as enabled in the target deployment.

3. ~~**Flavour-to-resource-class mapping:**~~ Resolved — the resource class is read directly from the flavour's `resources:CUSTOM_*` extra spec. No separate mapping needed.

4. **Placement ownership:** Who owns the lifecycle of a Placement — is it created and deleted by the region service on behalf of a user, or is it a user-visible resource? For the minimal service, treating it as an internal implementation detail of the region service (not directly user-facing) is simplest.

5. **Service placement:** Should the reservation service be embedded in `uni-region` (Option A) or deployed as a separate service `uni-reservation` (Option B)? See the Service Placement section. The deciding factor is likely host selection: if a "list available hosts for flavour" endpoint can be added to the region service, Option B becomes more viable. If not, Option A avoids duplicating Ironic and flavour-mapping logic.

6. **Rack membership data source for topology-aware host selection:** When topology constraints are added to Placements, the host selection algorithm must group candidate hosts by rack. The OpenStack Placement API returns individual resource providers (Ironic node UUIDs) with no rack grouping. A source of rack membership per node is needed — e.g. Ironic node traits, nested resource providers in Placement, or an operator-maintained topology map. This needs to be confirmed against the target deployment before topology-aware selection can be designed.
