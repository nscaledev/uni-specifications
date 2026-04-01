# Minimal Reservation Service — Design

## Overview

The reservation service pre-allocates bare metal hosts for a VPC before any server is booted. It is OpenStack-specific: it talks to Nova and Ironic, and its internal CRDs model OpenStack concepts directly.

**The problem it solves.** Bare metal host selection by Nova placement is opportunistic — at `nova boot` time, placement picks whatever host satisfies the flavour constraints. For workloads that need pre-boot infrastructure programming (IB partition assignment, network programming, topology-aware placement), the host must be known and programmed before the boot call. The reservation service pins the host early and holds it, allowing dependent services to complete their programming against a stable host identity before any instance runs.

**What the reservation service does.** A caller creates a Reservation specifying a region, a flavour, and a set of readiness gates. The controller selects an available host of the right flavour, creates a Nova host aggregate containing that host, and records the aggregate ID in status. External services (e.g. ib-manager) receive a notification, perform their per-host programming, and signal readiness by setting a named condition on the Reservation via the reservation service REST API. Once all gates are satisfied and the Reservation is marked Ready, a caller can create a Server against it: the region service reads the aggregate ID from the Reservation and passes it to Nova as a scheduler hint, constraining placement to the pre-programmed host.

**Scope.** This design covers the minimal case: one host per Reservation. Multi-host reservations (e.g. a full rack) are a planned expansion but not designed here.

---

## Design

### CRD: `Reservation`

The `Reservation` CRD is internal to the reservation service. Its schema uses OpenStack terminology directly — there is no provider abstraction.

```go
type ReservationSpec struct {
    // Region in which to allocate a host
    RegionID string
    // VPC (Network) this Reservation is associated with.
    // External services (e.g. ib-manager) use this to determine which
    // partition or network resource to program for the reserved host.
    NetworkID string
    // Flavour determining which class of host to select
    FlavorID string
    // Condition types that must be True before this Reservation is usable.
    // Copied verbatim from Region.Spec.ReadinessGates at creation time.
    ReadinessGates []string
}

type ReservationStatus struct {
    // Nova host aggregate UUID created for this Reservation
    AggregateID string
    // Ironic node UUIDs of the hosts assigned to this Reservation.
    // Set by the controller after host selection; read by external services
    // (e.g. ib-manager) to discover which hosts to program.
    HostIDs []string
    // Standard Kubernetes conditions.
    // Includes the controller-owned Ready condition and any gate conditions
    // set by external services.
    Conditions []metav1.Condition
}
```

**Conditions:**
- `Ready` — set True by the controller once the aggregate is created and hosts are assigned. This is the controller's own health signal, separate from the readiness gates.
- Gate conditions (e.g. `ib.unikorn-cloud.org/partition-ready`) — set by external services via the REST API. The region service waits for all conditions named in `Spec.ReadinessGates` to be True before using this Reservation as a scheduler hint.

A Reservation is usable when both `Ready == True` and all `Spec.ReadinessGates` conditions are True.

### Controller

On creation (DeletionTimestamp nil):

```
1. Select an available host
       Query Ironic for nodes with a resource class matching FlavorID
       Filter out nodes already referenced in another active Reservation
       Pick one node

2. Create Nova host aggregate
       POST /compute/v2/os-aggregates
       Add the selected host to the aggregate

3. Record in status
       Status.AggregateID = aggregate UUID
       Status.HostIDs = [ironic node UUID]
       Set Ready condition True
```

On deletion (DeletionTimestamp non-nil):

```
1. If GetResourceReferences() is non-empty: requeue, do not proceed
   (inbound references signal that a dependent service has not yet
   reversed its programming)

2. Delete Nova host aggregate
       DELETE /compute/v2/os-aggregates/{id}

3. Remove controller finalizer — Reservation is deleted
```

The controller adds its own finalizer on creation to ensure the deletion path runs before the object is removed from etcd.

### Host Selection

A host is considered available if:
- Its Ironic provision state is `available`
- It is not in maintenance
- Its resource class matches the requested flavour
- Its Ironic node UUID does not appear in `Status.HostIDs` of any other active (non-deleting) Reservation in the same region

Host selection is best-effort at creation time. If no host is available, the controller sets a condition (`HostUnavailable`) and requeues. It does not hold a lock between selection and aggregate creation — in the unlikely event of a race, the duplicate will be caught when the second reservation attempts to programme the same host (e.g. ib-manager will error on a conflicting GUID). The minimal service accepts this: single-host reservations with human-driven scheduling make races rare in practice.

### Resource References

External services that perform per-host programming register a deletion block on the Reservation while their programming is active. This prevents the aggregate from being torn down before the programming is reversed.

Per the platform spec (section 5.3), references cross service boundaries via the owning service's reference REST API — external services never write another service's finalizers directly. The reservation service exposes `PUT` and `DELETE` reference endpoints; internally it translates these into finalizer operations on its own CRD. The reference string format follows the canonical `{resource}.{group}/{uuid}` scheme produced by `GenerateResourceReference`.

**Lifecycle:**

```
ib-manager programs IB partition
    → PUT .../reservations/{id}/references/{ref}
    → reservation service adds finalizer to Reservation CRD

Reservation deleted (DeletionTimestamp set)
    → controller sees inbound reference via GetResourceReferences(), requeues

ib-manager receives deletion event, reverses UFM programming
    → DELETE .../reservations/{id}/references/{ref}
    → reservation service removes finalizer from Reservation CRD

No references remain
    → controller proceeds: deletes aggregate, removes its own finalizer
```

### Deletion Protocol

Full end-to-end deletion sequence:

```
User deletes Reservation
    │ DeletionTimestamp set
    ▼
Event bus fires → external services receive deletion notification
    ib-manager:
        Removes host GUIDs from UFM partition
        DELETE .../reservations/{id}/references/{ref}

Reservation controller (requeued):
    GetResourceReferences() non-empty?
        Yes → requeue
        No  → DELETE Nova aggregate
              Remove controller finalizer
              Reservation removed from etcd
```

---

## REST API

All endpoints are called by external services (region service, ib-manager). The reservation service writes to its own CRDs in response; callers never touch CRDs directly.

| Method | Path | Caller | Purpose |
|---|---|---|---|
| `POST` | `/reservations` | Region service | Create a Reservation |
| `GET` | `/reservations/{id}` | Region service, ib-manager | Read full Reservation state (host list, aggregate ID, conditions) |
| `DELETE` | `/reservations/{id}` | Region service | Delete a Reservation |
| `PUT` | `/reservations/{id}/conditions/{type}` | ib-manager | Set a named condition (e.g. `ib.unikorn-cloud.org/partition-ready: True`) |
| `PUT` | `/reservations/{id}/references/{name}` | ib-manager | Register a deletion block (idempotent) |
| `DELETE` | `/reservations/{id}/references/{name}` | ib-manager | Release a deletion block (idempotent) |

All calls use mTLS; the caller's service account certificate is its identity.

---

## Integration with the Region Service

### Creating a Reservation

The region service creates a Reservation when a Network is provisioned in an IB-capable region (or more generally, whenever the Region's `Spec.ReadinessGates` is non-empty). The Reservation is created with:
- `RegionID` from the Network's region
- `NetworkID` from the Network being provisioned
- `FlavorID` from the intended Server flavour
- `ReadinessGates` copied verbatim from `Region.Spec.ReadinessGates`

### Booting a Server

```
Region service creates Server

    1. Fetch Reservation via reservation service REST API
    2. Check Ready == True (aggregate created, host assigned)
    3. Check all Spec.ReadinessGates conditions are True
           If any are False or missing: Server stays pending, requeue

    4. All gates satisfied:
           Read Status.AggregateID from Reservation
           POST /compute/v2/servers with scheduler hint:
               os:scheduler_hints:
                 aggregate_instance_extra_specs:
                   unikorn-reservation: {reservation-id}
           (Aggregate metadata key set by reservation service at aggregate creation time)
```

Nova placement selects the single host within the aggregate. Because only one host is in the aggregate and all pre-boot programming is complete, placement is deterministic.

### Server-to-Reservation relationship

In the minimal service, one Reservation is consumed by at most one Server. The region service is responsible for enforcing this: once a Server has been booted against a Reservation, the Reservation is considered consumed and should not be reused.

---

## Integration with ib-manager

ib-manager subscribes to Reservation lifecycle events via the Kubernetes event bus (informer on the Reservation CRD). The envelope carries `ResourceID` and `DeletionTimestamp`.

**On creation (DeletionTimestamp nil):**
1. `GET /reservations/{id}` — fetch full state including `Spec.NetworkID` and `Status.HostIDs`
2. Fetch flavour via region service REST API → `FlavorInfiniBandSpec.PortCount`
3. If `PortCount == 0`: set `ib.unikorn-cloud.org/partition-ready` True — done
4. Look up the `IBPartition` for `Spec.NetworkID`
5. For each host in `Status.HostIDs`, query Ironic for IB port GUIDs
6. Program UFM partition (add GUIDs to the `IBPartition`)
7. `PUT /reservations/{id}/references/{ref}` — register deletion block
8. `PUT /reservations/{id}/conditions/ib.unikorn-cloud.org/partition-ready` True

**On deletion (DeletionTimestamp non-nil):**
1. `GET /reservations/{id}` — fetch current state
2. Remove host GUIDs from UFM partition
3. `DELETE /reservations/{id}/references/{ref}` — release deletion block

---

## Event Bus

The reservation service publishes Reservation lifecycle events to the Kubernetes event bus using the same envelope pattern as the region service: `ResourceID` and `DeletionTimestamp`. Subscribers (ib-manager) wire up a Kubernetes informer on the Reservation CRD type.

`DeletionTimestamp == nil` → creation or update event.
`DeletionTimestamp` non-nil → deletion in progress; subscriber should reverse its programming.

---

## Future Expansion

The minimal design is deliberately constrained to one host per Reservation. The intended expansion path:

- Add `Spec.Count int` (defaulting to 1) to support multi-host reservations
- `Status.HostIDs` already a slice — no schema change needed for the multi-host case
- Host selection logic generalises from "pick one" to "pick N"
- Resource reference and gate mechanisms are unchanged
- Further expansion to rack-level reservations would add a `Spec.Topology` field describing the grouping constraint (e.g. all hosts on the same top-of-rack switch)

---

## Decision Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-04-01 | `Spec.NetworkID` carried on Reservation | External services (e.g. ib-manager) need the VPC association to know which partition or network resource to program. Carrying it in Spec avoids a separate lookup and makes the Reservation self-describing. |
| 2026-04-01 | Deletion blocking via reference REST API, not direct finalizer writes | Per platform spec section 5.3, external services never write another service's finalizers directly. The reservation service exposes canonical `PUT`/`DELETE` reference endpoints and manages finalizers internally. |

---

## Service Placement

Two deployment options exist. The choice has not yet been made.

### Option A: Embedded in the Region Service

The reservation controller and handler live inside `uni-region`. The `Reservation` CRD belongs to the region service API group.

**Advantages:**
- Host selection can use internal region service state directly — flavour-to-resource-class mapping, Ironic client, and existing host inventory are all in-process. No new endpoints needed.
- One less service to deploy and operate.
- The Reservation is a natural extension of region concepts (flavours, hosts, aggregates).

**Disadvantages:**
- The region service grows larger. Reservation lifecycle (host selection, aggregate management, gate coordination) is a distinct concern from network and identity management.
- All consumers (ib-manager, future gate services) already call the region service; adding reservation endpoints there conflates two separate responsibilities.

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

1. **Host selection race:** The minimal service accepts a best-effort selection with no distributed lock. Is this acceptable for the target deployment, or does the host pool turn over fast enough that races are a real concern?

2. **Aggregate scheduler hint mechanism:** The design uses `aggregate_instance_extra_specs` with a `unikorn-reservation` metadata key on the aggregate, requiring `AggregateInstanceExtraSpecsFilter` in Nova's scheduler filter list. This needs to be confirmed as enabled in the target deployment.

3. **Flavour-to-resource-class mapping:** Host selection requires mapping `FlavorID` to an Ironic resource class. The mechanism for this lookup (via region service, directly from Nova, or via a local cache) is not yet specified.

4. **Reservation ownership:** Who owns the lifecycle of a Reservation — is it created and deleted by the region service on behalf of a user, or is it a user-visible resource? For the minimal service, treating it as an internal implementation detail of the region service (not directly user-facing) is simplest.

5. **Service placement:** Should the reservation service be embedded in `uni-region` (Option A) or deployed as a separate service `uni-reservation` (Option B)? See the Service Placement section. The deciding factor is likely host selection: if a "list available hosts for flavour" endpoint can be added to the region service, Option B becomes more viable. If not, Option A avoids duplicating Ironic and flavour-mapping logic.
