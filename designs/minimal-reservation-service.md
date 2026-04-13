# Minimal Reservation Service — Design

## Overview

The reservation service pre-allocates bare metal hosts before any server is booted. The design here is OpenStack-specific: it talks to Nova and Ironic, and its internal CRDs model OpenStack concepts directly.

**The problem it solves.**

Bare metal host selection by Nova placement is opportunistic — at `nova boot` time, placement picks whatever host satisfies the flavour constraints. For workloads that need pre-boot infrastructure programming (IB partition assignment, network programming, topology-aware placement), the host must be known and programmed before the boot call. The reservation service pins the hosts early and holds them, allowing dependent services to complete their programming against a stable host identity before any instance runs.

In addition, we want primitives for placing a set of hosts for a cluster, so they have optimal connectivity or resilience.

**What the reservation service does.**

The design uses two abstractions with separate concerns.

A *Reservation* is a capacity boundary at the NVLink domain level (for simplicity, with some loss of accuracy: "rack"). It claims a set of racks exclusively for a project or purpose, preventing those racks from being used by any other Reservation.

A *Placement* is an allocation carved from a Reservation. At allocation time, the controller selects a specific set of hosts from the Reservation's racks, applying spread constraints to control how those hosts are distributed. It refers to a VPC and is represented in OpenStack as a Nova host aggregate. The host list is fixed once the Placement is confirmed.

Server creation is then unconditional fill: the topology work is done at Placement creation. The region service reads the aggregate ID from the Placement and passes it to Nova as a scheduler hint; no host selection or spread logic runs at server boot time. Any additional preparation (e.g., IB partition assignment) is already programmed before the first server boots.

**Scope.** This design covers the two-level model (Reservation + Placement), multi-host Placements with topology spread constraints, and the gate-based readiness protocol used to support additional host preparation.

---

## Design

### Physical Hierarchy

```
Site
  └── POD (a.k.a., scaling unit; everything connected within one hop)
        └── Rack or NVLink domain (e.g., for GB300 NVL72 deployments, it's the whole rack; for a B200 deployment it coincides with hosts)
              └── Machine (one compute tray)
```

For a site, these are fixed and homogeneous — every POD contains the same number of racks, every rack the same number of machines. Capacity at any level is derivable by multiplication.

For GB300, the minimum Reservation granularity is **one full rack**: the NVLink domain boundary means a single GB300 compute tray triggers reservation of the entire rack.

### CRD: `OpenStackReservation`

The `OpenStackReservation` CRD is internal to the reservation service. It represents a capacity boundary at rack granularity; it contains no host list, no VPC, and no IB partition.

```go
type OpenStackReservationRequestTopology struct {
    // Hierarchy level to reserve at (e.g. "rack").
    Level string
    // Number of units at that level.
    Count int
}

type OpenStackReservationRequestResources struct {
    GPUCount int
    GPUModel string
}

type OpenStackReservationRequest struct {
    // Style is "topology-first" or "resource-first".
    Style string
    // Topology is set when Style is "topology-first".
    Topology *OpenStackReservationRequestTopology
    // Resources is set when Style is "resource-first".
    Resources *OpenStackReservationRequestResources
    // FailurePolicy controls behaviour when exact satisfaction is impossible.
    // "hard" rejects; "soft" allocates as much as possible.
    FailurePolicy string
}

type OpenStackReservationLocalityPreference struct {
    // Level is the hierarchy level to prefer locality within (e.g. "scale_unit").
    Level string
    // FailurePolicy controls behaviour if locality cannot be achieved.
    FailurePolicy string
}

type OpenStackReservationSpec struct {
    // Region in which to reserve capacity.
    RegionID string
    // Request describes how much capacity to claim and at what level.
    Request OpenStackReservationRequest
    // LocalityPreference optionally constrains rack selection to minimise
    // the number of scale units spanned.
    LocalityPreference *OpenStackReservationLocalityPreference
    // NotBefore is a schema stub; no automated transitions yet.
    NotBefore *metav1.Time
    // ExpiresAt is a schema stub; no automated transitions yet.
    ExpiresAt *metav1.Time
    // ParentReservationID is a schema stub for nested reservations.
    ParentReservationID string
}

type OpenStackReservationStatus struct {
    // RackIDs is the confirmed set of Ironic node UUIDs for the reserved racks.
    // Set atomically at creation; immutable thereafter.
    RackIDs []string
    // LocalityLevel is the highest hierarchy level at which locality was achieved.
    LocalityLevel string
    // State is "Active" or "Deleting".
    State string
}
```

### Reservation Controller

On creation (DeletionTimestamp nil):

```
1. Validate the request against the hierarchy configuration.
   For resource-first: compute minimum rack count to satisfy the resource
   request, then proceed as topology-first.

2. Query racks with sufficient healthy machines not already covered
   by any active Reservation.

3. Apply locality preference: prefer racks within the fewest scale units.
   If failure_policy is "hard" and a single scale unit cannot supply
   the requested count, reject with a clean error.

4. Conflict-check: no rack in the candidate set appears in any existing
   active Reservation.

5. Atomically write Status.RackIDs, Status.LocalityLevel, Status.State = Active.
   Add controller finalizer.
```

On deletion (DeletionTimestamp non-nil):

```
1. If any Placement with this ReservationID still exists: requeue.

2. Clear Status.RackIDs, set Status.State = Deleting.

3. Remove controller finalizer — Reservation is deleted.
```

### CRD: `OpenStackPlacement`

The `OpenStackPlacement` CRD is internal to the reservation service. Its schema uses OpenStack terminology directly — there is no provider abstraction.

```go
type PlacementSpreadConstraint struct {
    // Level is the hierarchy level to spread across (e.g. "rack").
    Level string
    // MaxSkew is the maximum permitted difference in host count between
    // any two nodes at this level. 0 means as even as possible.
    MaxSkew int
    // FailurePolicy controls behaviour when the constraint cannot be satisfied.
    // "hard" rejects; "soft" records the actual skew and proceeds.
    FailurePolicy string
}

type PlacementSpreadResult struct {
    Level         string
    RequestedSkew int
    ActualSkew    int
    Satisfied     bool
}

type OpenStackPlacementSpec struct {
    // Region in which to allocate hosts.
    RegionID string
    // ReservationID is the Reservation from which hosts are drawn.
    // Host selection is constrained to the Reservation's RackIDs.
    ReservationID string
    // MachineCount is the number of hosts required.
    MachineCount int
    // SpreadConstraints controls how MachineCount hosts are distributed
    // across topology levels within the Reservation's racks.
    SpreadConstraints []PlacementSpreadConstraint
    // NetworkID (VPC) this Placement is associated with.
    // Gate services use this to determine which network resource to program
    // for the placed hosts.
    NetworkID string
    // FlavorID determines which class of host to select.
    FlavorID string
    // ReadinessGates lists condition types that must be True before this
    // Placement is usable. Copied verbatim from Region.Spec.ReadinessGates
    // at creation time.
    ReadinessGates []string
}

type OpenStackPlacementStatus struct {
    // Nova host aggregate UUID created for this Placement.
    // Also serves as the disjointness signal: hosts in this aggregate
    // are ineligible for selection by any other Placement.
    AggregateID string
    // Ironic node UUIDs of the hosts assigned to this Placement.
    // Set atomically at creation; immutable thereafter.
    // Read by gate services to discover which hosts to program.
    HostIDs []string
    // SpreadResults records per-constraint outcomes: actual skew achieved
    // and whether the failure policy was honoured.
    SpreadResults []PlacementSpreadResult
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

### Placement Controller

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
       Status.SpreadResults = [per-constraint outcomes]
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

The controller loads the parent Reservation to constrain the candidate set, then queries the OpenStack Placement API, then applies spread constraints.

**Step 0 — Load the parent Reservation.**
Verify `Status.State == Active`. Collect `Status.RackIDs`. All subsequent candidate enumeration is restricted to hosts whose rack membership is within this set.

**Step 1 — Resolve the resource class from the flavour.**
The flavour's extra specs contain `resources:CUSTOM_BAREMETAL_LARGE=1` (or equivalent). The controller queries Nova for the flavour and reads the `resources:CUSTOM_*` key to obtain the resource class name.

**Step 2 — Query Placement for available hosts.**

```
GET /placement/resource_providers?resources=CUSTOM_BAREMETAL_LARGE:1
```

This returns only resource providers with free capacity. This is the correct source of truth because a node mid-deploy is already marked consumed in Placement even though its Ironic provision state is still `deploying`. The result is filtered to hosts whose rack membership is within the Reservation's `RackIDs`.

**Step 3 — Filter by aggregate membership.**
Each candidate is checked against Nova: any host already a member of an existing Placement aggregate is excluded. This is the disjointness mechanism — it ensures a host cannot be assigned to more than one Placement without requiring cross-CRD scanning.

**Step 4 — Cross-check against Ironic.**
Placement has a ~60-second sync lag from Ironic. To guard against stale data, each remaining candidate is checked directly against Ironic:
- `provision_state == available`
- `maintenance == false`

**Step 5 — Apply spread constraints.**
Group the remaining candidates by rack. For each constraint in `Spec.SpreadConstraints`, distribute `MachineCount` hosts to satisfy `MaxSkew`. If `FailurePolicy: hard` and the constraint cannot be satisfied with available hosts, reject and return an error. Record actual skew per constraint in `SpreadResults`.

If no sufficient set of hosts is available, the controller sets a `HostUnavailable` condition and requeues. Host selection is best-effort with no distributed lock — in the event of a race between two concurrent Placements selecting the same host, the aggregate-membership check reduces but does not eliminate the window.

### Resource References

External services that perform per-host programming register a deletion block on the Placement while their programming is active. This prevents the aggregate from being torn down before the programming is reversed.

Per the platform spec (section 5.3), references cross service boundaries via the owning service's reference REST API — external services never write another service's finalizers directly. The reservation service exposes `PUT` and `DELETE` reference endpoints; internally it translates these into finalizer operations on its own CRD. The reference string format follows the canonical `{resource}.{group}/{uuid}` scheme produced by `GenerateResourceReference`.

**Lifecycle:**

```
Gate service performs per-host programming
    → PUT .../placements/{id}/references/{ref}
    → reservation service adds finalizer to Placement CRD

Placement deleted (DeletionTimestamp set)
    → controller sees inbound reference via GetResourceReferences(), requeues

Gate service receives deletion event, reverses programming
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
Event bus fires → gate services receive deletion notification
    gate service:
        Reverses per-host programming
        DELETE .../placements/{id}/references/{ref}

Placement controller (requeued):
    GetResourceReferences() non-empty?
        Yes → requeue
        No  → DELETE Nova aggregate
              Remove controller finalizer
              Placement removed from etcd

All Placements from Reservation deleted
    │
    ▼
User deletes Reservation
    → Reservation controller releases RackIDs, removes finalizer
```

---

## REST API

All endpoints are called by external services (region service, gate services). The reservation service writes to its own CRDs in response; callers never touch CRDs directly.

| Method | Path | Caller | Purpose |
|---|---|---|---|
| `GET` | `/topology` | Region service | Hierarchy config and resource profile per machine |
| `GET` | `/topology/nodes` | Region service | All nodes at a given level with available count |
| `POST` | `/reservations` | Region service / operator | Create a Reservation |
| `GET` | `/reservations` | Region service | List active Reservations for the project |
| `GET` | `/reservations/{id}` | Region service | Read Reservation state and rack IDs |
| `DELETE` | `/reservations/{id}` | Region service / operator | Delete a Reservation |
| `POST` | `/placements` | Region service | Create a Placement |
| `GET` | `/placements/{id}` | Region service, gate services | Read full Placement state (host list, aggregate ID, conditions) |
| `DELETE` | `/placements/{id}` | Region service | Delete a Placement |
| `PUT` | `/placements/{id}/conditions/{type}` | Gate service | Set a named condition |
| `PUT` | `/placements/{id}/references/{name}` | Gate service | Register a deletion block (idempotent) |
| `DELETE` | `/placements/{id}/references/{name}` | Gate service | Release a deletion block (idempotent) |

All calls use mTLS; the caller's service account certificate is its identity.

---

## Integration with the Region Service

### Creating a Reservation

The region service (or operator tooling) creates a Reservation to claim a set of racks for a project. Example for a 2-rack dev slice with soft scale-unit locality:

```json
POST /reservations
{
  "request": {
    "style": "topology-first",
    "topology": { "level": "rack", "count": 2 },
    "failure_policy": "hard"
  },
  "locality_preference": { "level": "scale_unit", "failure_policy": "soft" }
}
```

### Creating a Placement

The region service creates a Placement when a Network is provisioned in an IB-capable region. The Placement is created with:
- `RegionID` from the Network's region
- `ReservationID` of the Reservation covering this project's racks
- `MachineCount` from the intended cluster size
- `SpreadConstraints` from the cluster topology policy
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

Nova selects any available host within the aggregate. Because all pre-boot programming is complete and the IB partition covers the full host set, placement is deterministic. Concurrent server creation calls against the same Placement are safe — Nova handles contention within the aggregate.

---

## Integration with Gate Services

A gate service subscribes to Placement lifecycle events via the Kubernetes event bus (informer on the `OpenStackPlacement` CRD). The envelope carries `ResourceID` and `DeletionTimestamp`.

**On creation (DeletionTimestamp nil):**
1. `GET /placements/{id}` — fetch full state including `Spec.NetworkID` and `Status.HostIDs`
2. Determine whether this Placement requires programming by this gate service (e.g. check flavour capabilities).
3. If not applicable: set the gate condition True and return.
4. Perform per-host programming using `Spec.NetworkID` and `Status.HostIDs` as inputs.
5. `PUT /placements/{id}/references/{ref}` — register deletion block
6. `PUT /placements/{id}/conditions/{gate-condition}` True
   → reservation service controller requeues, recomputes Ready

**On deletion (DeletionTimestamp non-nil):**
1. `GET /placements/{id}` — fetch current state
2. Reverse per-host programming
3. `DELETE /placements/{id}/references/{ref}` — release deletion block

---

## Event Bus

The reservation service publishes Placement lifecycle events to the Kubernetes event bus using the same envelope pattern as the region service: `ResourceID` and `DeletionTimestamp`. Gate services wire up a Kubernetes informer on the `OpenStackPlacement` CRD type.

`DeletionTimestamp == nil` → creation or update event.
`DeletionTimestamp` non-nil → deletion in progress; subscriber should reverse its programming.

---

## Future Expansion

**Pre-booking (R3).** Activate `NotBefore` / `ExpiresAt` transitions on the Reservation. Add time-window overlap to conflict detection. No schema changes needed.

**Ranked fallback (R4).** A retry loop around the Reservation satisfaction algorithm, relaxing the rack count by one per iteration.

**Spot capacity.** Introduce a pool of machines not covered by any active Reservation. Spot Placements draw from this pool with no exclusivity guarantee.

**Nested Reservations.** Activate `ParentReservationID`. Enforce that child `RackIDs` ⊆ parent `RackIDs`.

**Topology labels on instances.** Hierarchy position labels fall directly out of `HostIDs` and the hierarchy model. Can be added to the server provisioner independently.

**Multiple Placements per Reservation.** Already supported by the model — a Reservation's `RackIDs` can be parcelled into multiple non-overlapping Placements. Conflict detection at the host level prevents overlap automatically.

---

## Decision Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-04-01 | `Spec.NetworkID` carried on Placement | Gate services need the VPC association to know which network resource to program. Carrying it in Spec avoids a separate lookup and makes the Placement self-describing. |
| 2026-04-01 | Deletion blocking via reference REST API, not direct finalizer writes | Per platform spec section 5.3, external services never write another service's finalizers directly. The reservation service exposes canonical `PUT`/`DELETE` reference endpoints and manages finalizers internally. |
| 2026-04-01 | CRD named `OpenStackPlacement` | The CRD is OpenStack-specific (Nova aggregates, Ironic node UUIDs). Making the provider explicit in the type name is consistent with the design's stated principle of no provider abstraction, and avoids naming conflicts if other provider types are added later. |
| 2026-04-01 | Host selection via Placement API, cross-checked against Ironic | `GET /placement/resource_providers?resources=<class>:1` gives a more accurate availability view than querying Ironic directly — it reflects Nova allocations for nodes mid-deploy. Resource class is resolved from the flavour's `resources:CUSTOM_*` extra spec. Ironic is still checked to guard against Placement's ~60s sync lag. |
| 2026-04-08 | Disjointness enforced via Nova aggregate membership | A host already in any Placement aggregate is excluded from selection by a new Placement. This uses Nova's own state as the source of truth rather than scanning CRDs, and provides the same guarantee with less coordination. |
| 2026-04-08 | Topology constraints are a Placement concern, not a Reservation concern | Topology is a placement policy — where to put hosts — not a capacity policy. Rack affinity and spread are placement use cases. The Reservation layer expresses count and locality; the Placement expresses spread and host selection. |
| 2026-04-09 | Reservation is a rack-level capacity boundary; Placement carves hosts from it | Separates the operator concern (which racks are dedicated to this project) from the workload concern (how to distribute N hosts across those racks). Gate services' contract is unchanged — they still see only a Placement with a fixed host list, a VPC, and readiness gates. |
| 2026-04-09 | Server creation is unconditional aggregate fill | Spread constraints fire once at Placement creation. At server boot time there is no host selection and no spread logic — Nova picks any available host within the aggregate. This keeps the server provisioner simple and makes concurrent server creation safe. |

---

## Migration

The current system boots servers directly via Nova with no pre-placement, no Placements, and no readiness gates. The migration must be non-breaking for existing regions and existing servers.

### Existing regions — no change required

`Region.Spec.ReadinessGates` is a new optional field that defaults to empty. An empty gate list means no gates are required before a Placement is used as a scheduler hint — and with no Placement in the boot path at all, existing regions behave exactly as before. No operator action is needed for regions that do not have an IB fabric.

### New IB-capable regions — opt-in by operator

A region with an IB fabric is configured with `ib.unikorn-cloud.org/partition-ready` in `Region.Spec.ReadinessGates` at deploy time. Only these regions use Placements and gate-based pre-boot coordination. Enabling a gate service for a region is a single operator change to the Region definition; no flavour updates or server changes are needed.

### Existing servers — no backfill

Servers already running have no associated Placement. They do not need one: per-host programming for running servers was never performed (there were no gate services), and retrofitting a Placement to a running server would not change the hardware state. The Server provisioner must handle both cases:

- **No Placement reference on the Server:** use the existing direct Nova placement path (current behaviour, unchanged).
- **Placement reference present:** check all gates, then boot with the aggregate scheduler hint.

This conditional is the only place in the codebase where the old and new paths diverge. Existing servers continue to work without modification.

### Schema changes are additive

All CRD changes introduced by this design are additive optional fields:

| Resource | New field | Default | Impact on existing resources |
|---|---|---|---|
| `Region` | `Spec.ReadinessGates []string` | `[]` (empty) | None — existing Regions have no gates |
| `OpenStackReservation` | Entire new CRD | — | None — no existing Reservations |
| `OpenStackPlacement` | Entire new CRD | — | None — no existing Placements |
| `Flavor` | `Spec.InfiniBand.PortCount int` | `0` | None — existing Flavours have zero IB ports |

No data migration is required. Existing resources are valid under the new schema without modification.

### Rollout order

1. Deploy the updated region service with `Region.Spec.ReadinessGates` support and the conditional Server provisioner path.
2. Deploy the reservation service (or activate it within `uni-region` depending on the placement decision).
3. Deploy gate services.
4. For each region requiring gate-based pre-boot coordination, add the relevant gate condition names to `Region.Spec.ReadinessGates`.

Steps 1–3 have no user-visible effect. Step 4 activates the new path for that region only.

---

## Service Placement

Two deployment options exist. The choice has not yet been made.

### Option A: Embedded in the Region Service

The Placement and Reservation controllers and handlers live inside `uni-region`. The `OpenStackPlacement` and `OpenStackReservation` CRDs belong to the region service API group.

**Advantages:**
- Host selection can use internal region service state directly — flavour-to-resource-class mapping, Ironic client, and existing host inventory are all in-process. No new endpoints needed.
- One less service to deploy and operate.
- The Placement is a natural extension of region concepts (flavours, hosts, aggregates).

**Disadvantages:**
- The region service grows larger. Placement and Reservation lifecycle is a distinct concern from network and identity management.
- All consumers (gate services) already call the region service; adding placement endpoints there conflates two separate responsibilities.

### Option B: Separate Service (`uni-reservation`)

The reservation service is a standalone binary with its own CRDs, API, and deployment.

**Advantages:**
- Clean separation of concerns. The reservation service owns host pre-allocation; the region service owns networks and identities.
- Independently deployable and scalable.
- The service boundary makes the gate protocol explicit — gate services call `uni-reservation`, not `uni-region`.

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

6. **Rack membership data source (blocking):** The Reservation creation algorithm requires grouping candidate hosts by rack. The OpenStack Placement API returns individual resource providers (Ironic node UUIDs) with no rack grouping. A source of rack membership per node must be confirmed before the Reservation controller can be implemented — candidates: Ironic node traits, nested resource providers in OpenStack Placement, or an operator-maintained topology ConfigMap.

7. **Unreserved Placements:** The two-level model makes `ReservationID` a required field on a Placement. The previous design allowed Placements against the open pool. Should unreserved Placements remain permitted as a fallback, or is a Reservation now always required?
