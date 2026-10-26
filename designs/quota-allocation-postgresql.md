# Quota Service Extraction and PostgreSQL Implementation

## Summary

Six main steps, each independently deployable and reversible until
M5. M4 has substeps that can be landed incrementally:

- **M1** — Move quota handler logic into `pkg/quota/handler/` and
  expose it on a second HTTP listener within identity's
  process. Create the `uni-quota` repo with forward-compat shims
  pointing at `uni-identity`. Identity continues to serve all quota
  endpoints unchanged.
- **M2** — Port each consumer in its own PR: switch the quota URL to
  the new listener and update module imports to `uni-quota`. This is
  the only consumer-facing step in the migration.
- **M3** — Once all consumers have completed M2, remove quota
  endpoints from identity's spec and the delegation code from
  identity's handler. No compat shim needed — consumers are already on
  `uni-quota`.
- **M4** — Add PostgreSQL backend to the quota listener with
  dual-write to CRDs. (M4a: schema migration; M4b: data migration job;
  M4c: switch backend with startup sync and consistency controller.)
  Rollback is a flag flip; CRDs stay current via dual-write.
- **M5** — Drop CRD writes. PostgreSQL is now the sole store. Point of
  no return — rollback past here requires manual reconciliation.
- **M6** — Move quota code from `uni-identity` to `uni-quota`,
  replacing shims with real implementations, and extract
  `unikorn-quota` as a standalone binary. No consumer impact.

---

## Background

Quota limits and resource allocations are currently implemented inside the identity service.
They are served under the identity API, stored as Kubernetes CRDs, and consumed by other
services via a thin client wrapper (`pkg/client/allocations.go`) that calls back into identity.

They ended up there for convenience — identity already had organization/project scoping and
RBAC — but quota management is a separate concern. Keeping it inside identity means:

- The quota implementation cannot be changed without modifying identity.
- Other UNI services are coupled to identity for a function that has nothing to do with
  authentication or authorization.
- The current CRD-backed implementation has a correctness problem (see below) that cannot
  be fixed cleanly without introducing a database dependency into identity, which would
  break UNI's assumption that Kubernetes is the only required datastore.

### Correctness problem with the current implementation

Allocation create and update operations use an in-process `sync.Mutex` to serialize the
quota-check/write sequence. This only works within a single replica. Running more than one
identity pod introduces a TOCTOU race that can allow quota over-commitment. The mutex is
a workaround for the fact that etcd does not support the atomic check-and-reserve pattern
that this use case requires.

Additionally, the current code commits allocations in the HTTP handler, which violates the
two-phase reservation model specified in section 7.6 (documented as Appendix A.3 deficiency).

---

## Proposal

Extract quota and allocation management into a **standalone quota service** (`unikorn-quota`)
with its own OpenAPI spec, its own binary, and its own storage. Other services consume it
via a generated client pointed at a configurable URL.

The separation is at the **HTTP API level**, not at a Go interface level. `unikorn-quota`
supports two backends, selected by configuration:

- **CRD-backed** — the default, no database required, satisfies UNI's Kubernetes-only
  assumption.
- **PostgreSQL-backed** — fixes the multi-replica correctness problem.

Any service that speaks the quota API can be dropped in as a replacement. UNI deployments
never need to know PostgreSQL exists.

The extraction is done incrementally via the migration steps below, keeping identity
serving the quota endpoints until all consumers have moved. See the Migration section for
the full sequence.

---

## The Quota API

The quota endpoints are extracted from `pkg/openapi/server.spec.yaml` into a new
`pkg/quota/openapi/server.spec.yaml`. The endpoints themselves do not change:

```
GET  /api/v1/organizations/{organizationID}/quotas
PUT  /api/v1/organizations/{organizationID}/quotas

POST   /api/v1/organizations/{organizationID}/projects/{projectID}/allocations
GET    /api/v1/organizations/{organizationID}/projects/{projectID}/allocations/{allocationID}
PUT    /api/v1/organizations/{organizationID}/projects/{projectID}/allocations/{allocationID}
DELETE /api/v1/organizations/{organizationID}/projects/{projectID}/allocations/{allocationID}
```

A client package is generated from the new spec. The hand-written `pkg/client/allocations.go`
wrapper is retained permanently — it encapsulates useful logic shared by all consumers
(principal extraction, allocation ID annotation management, resource reference generation)
that would otherwise need to be duplicated. It is updated during the migration to use a
narrow interface (M1) and `quotaapi` types (M3), but not removed.

RBAC verification uses the standard UNI pattern: validate the bearer token against
identity, then enforce scopes locally. This is identical to how region, compute, and
other UNI services handle authorisation today.

---

## The PostgreSQL Backend

At M6, the quota service is extracted as a standalone binary. It supports both backends;
PostgreSQL is selected by configuration.

### Database schema

```sql
-- Per-organization, per-kind quota limits.
CREATE TABLE quota_limits (
    organization_id TEXT   NOT NULL,
    kind            TEXT   NOT NULL,
    quantity        BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (organization_id, kind)
);

-- One row per managed resource that holds an allocation.
-- id is the allocation UUID, preserved from the CRD object name during migration.
CREATE TABLE allocations (
    id                UUID  PRIMARY KEY,
    organization_id   TEXT  NOT NULL,
    project_id        TEXT  NOT NULL,
    resource_kind     TEXT  NOT NULL,
    resource_id       TEXT  NOT NULL,
    creator_id        TEXT,
    creator_principal TEXT,
    tags              JSONB,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, resource_kind, resource_id)
);

-- Per-quota-kind committed and reserved amounts within an allocation.
CREATE TABLE allocation_items (
    allocation_id UUID   NOT NULL REFERENCES allocations(id) ON DELETE CASCADE,
    kind          TEXT   NOT NULL,
    committed     BIGINT NOT NULL DEFAULT 0,
    reserved      BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (allocation_id, kind)
);
```

### Atomic check-and-reserve

The central operation — check quota, write allocation — is a single database transaction
with row-level locking:

```
BEGIN
  SELECT kind, quantity FROM quota_limits
    WHERE organization_id = $1
    FOR UPDATE                          -- blocks concurrent allocations for this org

  SELECT SUM(committed + reserved) ... -- aggregate current usage
  -- return 403 if new total would exceed any limit

  INSERT / UPDATE allocations + allocation_items
COMMIT
```

This replaces the `sync.Mutex` and is safe across any number of replicas.

### Accommodating the two-phase model (future)

The schema supports the spec section 7.6 two-phase model without further migration.
Add a `status` column:

```sql
ALTER TABLE allocations
  ADD COLUMN status TEXT NOT NULL DEFAULT 'committed'
      CHECK (status IN ('reserved', 'committed', 'releasing'));
```

- Handler creates row with `status = 'reserved'`.
- Controller sets `status = 'committed'` on provisioning.
- Controller sets `status = 'releasing'`, then deletes, on deprovisioning.
- Quota accounting sums both `reserved` and `committed` rows, ignoring `releasing`.

### QuotaMetadata

`QuotaMetadata` CRDs (display names, descriptions, defaults for each quota kind) are
platform configuration rather than operational data. The PostgreSQL implementation reads
them from the Kubernetes API the same way the CRD implementation does, or they can be
provided as static configuration. This decision does not affect the schema.

---

## Migration

The migration is broken into small, independently deployable steps following a
trunk-based approach. Each step has a well-understood rollback that does not require
data reconciliation, until the final step where CRD writes are intentionally dropped.

The key insight is that **shared storage eliminates data divergence during cutover**.
While both identity and the new quota service are backed by the same CRDs, rollback is
always a configuration change (re-point a URL), never a data operation.

Consumer services adopt the `uni-quota` module path at M2, the same step where they
switch their quota URL. Forward-compat shims in `uni-quota` (pointing at `uni-identity`)
mean this is a mechanical import-path change with no behavioural difference. Code
physically moves to `uni-quota` at M6, after M5, with no further consumer porting.

### Step M1: Add quota listener to identity, CRD-backed

Move quota and allocation handler logic into a new `pkg/quota/handler/` package and
expose it on a second HTTP listener within the identity process. Identity continues to
serve all endpoints it serves today. No consumers are affected.

#### Why not a separate binary

The CRD backend serialises allocation decisions with an in-process `sync.Mutex`. A
separate binary would have an independent mutex and reintroduce the TOCTOU race the
mutex exists to prevent: two processes could each pass their local quota check before
either commits a write. Running the quota handler as a second listener inside identity's
process allows it to share identity's mutex and `directclient` directly. `unikorn-quota`
can only become a separate binary once the PostgreSQL backend (M4c) replaces the mutex
with database-level locking.

#### Alternatives considered for serving a subset of the API

The generated `ServerInterface` for identity is monolithic — it requires all 52 methods
to be implemented. The quota listener only needs 6. Three approaches were considered:

**Embed `Unimplemented`** — the generated code includes an `Unimplemented` struct that
returns 405 for every method. The quota listener could embed it and override only the 6
quota methods. Simple, but all 52 routes are still registered in the chi router and
return 405, and the listener nominally "implements" endpoints it does not own.

**Split the spec and use two routers** — create a separate
`pkg/quota/openapi/server.spec.yaml` with only the 6 quota endpoints, generating a
clean 6-method `ServerInterface`. The identity listener uses the existing identity
router; the quota listener uses only the quota router. No zombie routes, no
`Unimplemented` embedding. **This is the chosen approach.**

**Custom chi router without generated interface** — register the 6 quota routes manually
on a chi router without using the generated `ServerInterface` at all. Loses the
type-safe parameter binding that code generation provides.

#### Spec split

A new `pkg/quota/openapi/server.spec.yaml` defines only the 6 quota and allocation
endpoints. Quota-specific types (`AllocationRead`, `AllocationWrite`, `QuotaRead`,
etc.) are defined in both specs for now. The oapi-codegen `import-mapping` configuration
for the quota spec should be tried first to reference these types from
`github.com/unikorn-cloud/identity/pkg/openapi` rather than redefining them; if
local-file `$ref` import-mapping is not supported by the tool, duplicating the
definitions in both specs is acceptable until M3.

#### Quota handler

A new `pkg/quota/handler/` package implements the 6-method quota `ServerInterface`
generated from the quota spec. Its constructor accepts the shared `*sync.Mutex` and
uncached `client.Client` as arguments, and delegates to the existing underlying client
code (`pkg/handler/quotas/client.go`, `pkg/handler/allocations/client.go`) — no logic
is duplicated. This package is the canonical quota handler going forward; identity's
`Handler` quota and allocation methods are refactored to delegate to it, and the quota
listener uses it directly.

#### Two listeners, one process

Identity's `main.go` starts two HTTP servers on separate ports:

- The identity listener serves all 52 endpoints via identity's `Handler`, whose 6 quota
  and allocation methods now delegate to `pkg/quota/handler/`.
- The quota listener serves only the 6 quota and allocation endpoints via
  `pkg/quota/handler/`.

Both are constructed with the same `*sync.Mutex` and `directclient`, so allocation
decisions are fully serialised across both listeners. A new Kubernetes Service exposes
the quota listener port; this is the URL consumers will point at in M2.

#### uni-quota forward-compat shims

A new `uni-quota` repository is created alongside the listener. It immediately provides
forward-compatibility shims so that consumers can adopt the `uni-quota` module path at
M2 before any code physically moves there:

- `uni-quota/pkg/openapi` — re-exports quota types from `uni-identity/pkg/quota/openapi`
  using type aliases.
- `uni-quota/pkg/client` — re-exports `NewAllocations` and the `Allocations` type from
  `uni-identity/pkg/client`.

`uni-quota` depends on `uni-identity` for these re-exports; `uni-identity` does not
depend on `uni-quota`. The dependency runs in the correct direction throughout M1–M5.
When code physically moves to `uni-quota` at M6, the shims are replaced with real
implementations and the re-export dependency is removed.

A `builder.go` for `uni-quota/pkg/openapi` — analogous to the existing
`pkg/openapi/builder.go` in `uni-identity` — is also part of M1, so consumers can
construct a quota `BaseClient` using only `uni-quota` imports.

#### Narrow client interface

`pkg/client/allocations.go` is changed to accept a local 3-method interface rather than
the full `openapi.ClientWithResponsesInterface`:

```go
type allocationAPI interface {
    PostApiV1OrganizationsOrganizationIDProjectsProjectIDAllocationsWithResponse(...)
    PutApiV1OrganizationsOrganizationIDProjectsProjectIDAllocationsAllocationIDWithResponse(...)
    DeleteApiV1OrganizationsOrganizationIDProjectsProjectIDAllocationsAllocationIDWithResponse(...)
}
```

This decouples the wrapper from the specific generated package. The change is
backward-compatible: the existing `openapi.ClientWithResponsesInterface` satisfies the
narrower interface, so consumer call sites do not need to change at M1. The
quota-generated `ClientWithResponses` will also satisfy it, which is what consumers
switch to in M2.

Rollback: remove the quota listener configuration and its Kubernetes Service. No data impact.

### Step M2: Port consumers one at a time

Three services currently use allocation APIs: kubernetes, compute, and region. Each is
ported in its own PR. This is the only consumer-facing step in the entire migration.

Each PR makes five changes:

**1. Add quota endpoint configuration.**
A new `--quota-endpoint` flag is added alongside the existing `--identity-endpoint`. A
quota `BaseClient` is constructed the same way as the identity one, using the
`uni-quota` builder:

```go
baseclient.APIClient(ctx, quotaBase, uniqapi.NewBuilder())
```

This reuses the existing `BaseClient` infrastructure (token injection, TLS, etc.).

**2. Add `uni-quota` to `go.mod`.**
The consumer adds `github.com/unikorn-cloud/uni-quota` as a module dependency.

**3. Switch module imports.**
Allocation type imports are updated from `uni-identity/pkg/openapi` to
`uni-quota/pkg/openapi`, and the `NewAllocations` import from `uni-identity/pkg/client`
to `uni-quota/pkg/client`. The shims in `uni-quota` make this a mechanical path change
with no behavioural difference — the underlying types and implementation are identical.

**4. Add quota client to the handler and switch the `NewAllocations` call.**
The handler struct gets a dedicated `quota allocationAPI` field. The identity client
remains for all other identity operations. `NewAllocations` is called with the quota
client:

```go
// before
identityclient.NewAllocations(c.client, c.identity).Create(ctx, cluster, allocations)

// after
uniqclient.NewAllocations(c.client, c.quota).Create(ctx, cluster, allocations)
```

**5. Move Pact contract tests.**
Allocation contract tests currently live in e.g.
`kubernetes/test/contracts/consumer/identity/`. They move to
`kubernetes/test/contracts/consumer/quota/`, pointing at the quota service as the
provider. This ensures the quota service is tested to the same contract standard as
identity was.

Each consumer can be migrated independently. Rollback is reverting the PR; the service
re-points at identity which continues to serve quota endpoints until M3. Because both
services write to the same CRDs, quota state is consistent regardless of which endpoint
a consumer uses at any given time. The `uni-quota` shims mean there is no type
incompatibility on rollback.

### Step M3: Remove quota endpoints from identity

Once all consumers have completed M2, the quota and allocation endpoints are removed
from identity in a single PR. No compat shim is needed — consumers are already
importing from `uni-quota`.

**Identity spec and generated code.** The 6 quota/allocation endpoint definitions and
their associated type definitions are removed from `pkg/openapi/server.spec.yaml`. The
generated files (`types.go`, `router.go`, `client.go`, `schema.go`) are regenerated.

**Handler delegation removal.** The following are removed from identity:

- The quota and allocation delegation methods from `pkg/handler/handler.go` and
  `pkg/handler/handler_allocations.go` (the `ServerInterface` no longer requires them).
- The `pkg/quota/handler/` reference from identity's `Handler` struct.

`pkg/handler/quotas/`, `pkg/handler/allocations/`, and `pkg/quota/handler/` are
unaffected — they continue to serve the quota listener and will move to `uni-quota`
at M6. `allocationMutex` and `directclient` were already moved into `pkg/quota/handler/`
at M1 and are not touched here.

**`pkg/client/allocations.go`.** This wrapper is kept permanently. Its internal type
references are updated from `identityapi` to `quotaapi` in this PR, now that the
`identityapi` quota types have been removed from the generated code.

Rollback: revert the identity PR. Consumers continue working because the quota listener
is still running.

### Step M4a: Schema migration

A `cmd/unikorn-quota/` binary is added to this repository. Initially it contains only
the schema migration tooling (e.g. `golang-migrate`) and the initial schema SQL. A
pre-upgrade Helm Job runs it on each upgrade before any pods are rolled:

```yaml
metadata:
  annotations:
    helm.sh/hook: pre-upgrade
    helm.sh/hook-weight: "-10"           # runs before the data migration job
    helm.sh/hook-delete-policy: before-hook-creation
```

No behavioral change — the tables exist but nothing reads or writes them yet.

Rollback: the empty tables are harmless; no action required.

### Step M4b: Data migration Job

A second pre-upgrade Helm Job bulk-upserts existing CRD data into PostgreSQL. It runs
after the schema job (higher hook weight) and is idempotent and safe to re-run. The
service remains CRD-backed; PostgreSQL is populated but not yet used.

```yaml
metadata:
  annotations:
    helm.sh/hook: pre-upgrade
    helm.sh/hook-weight: "-5"
    helm.sh/hook-delete-policy: before-hook-creation
```

The migration logic:

```
for each Quota CRD across all namespaces:
  org_id = labels["unikorn-cloud.org/organization"]
  for each ResourceQuota in spec.quotas:
    UPSERT INTO quota_limits (organization_id, kind, quantity)
    ON CONFLICT (organization_id, kind)
      DO UPDATE SET quantity = excluded.quantity

for each Allocation CRD across all namespaces:
  UPSERT INTO allocations
    (id, organization_id, project_id, resource_kind, resource_id, ...)
    VALUES (allocation.Name, ...)       -- preserve UUID from CRD object name
  ON CONFLICT (id) DO NOTHING           -- idempotent: safe to re-run

  for each ResourceAllocation in spec.allocations:
    UPSERT INTO allocation_items (allocation_id, kind, committed, reserved)
    ON CONFLICT (allocation_id, kind)
      DO UPDATE SET committed = excluded.committed, reserved = excluded.reserved
```

Allocation IDs are preserved exactly from `Allocation.Name`. Resource annotations
(`unikorn-cloud.org/allocation`) on managed resources remain valid with no patching.

**Open question:** The `allocations` table has `creator_id` and `creator_principal`
columns. What labels or annotations on the Allocation CRD carry these values, and what
is the mapping? This needs to be resolved before implementing the migration job.

**Open question:** The migration job uses `ON CONFLICT (id) DO NOTHING` for allocation
rows, so it never removes PostgreSQL rows for CRDs that have been deleted since the last
run. If an allocation is deleted during the upgrade window (after M4b runs but before
M4c pods become ready), the M4c startup sync (PostgreSQL→CRD) will recreate the deleted
CRD. The window is narrow but the behaviour is surprising. Is this acceptable, or should
the job reconcile deletions?

M4a and M4b can land several releases before M4c, giving confidence that the migration
logic is correct before the backend switch matters.

Rollback: the populated tables are harmless; the service ignores them.

### Step M4c: Switch to PostgreSQL backend with dual-write

The quota listener switches to reading and writing PostgreSQL when `--db-url` is set. The
CRD-backed path remains available for deployments without PostgreSQL.

**Dual-write.** Writes go to PostgreSQL first (for atomic check-and-reserve — see
below), then to CRDs as a rollback safety net. CRD writes are retried with a short
backoff (3 attempts) since most failures are transient (optimistic lock conflict, brief
API unavailability). If retries are exhausted, the service logs at ERROR level with the
affected allocation or quota ID and continues — PostgreSQL is still correct and the
service is operational, but the CRD is stale.

**M4b job retired.** The M4b pre-upgrade job (CRD→PostgreSQL) is removed in this step.
With PostgreSQL now the source of truth, running a CRD→PostgreSQL sync could overwrite
correct data with stale CRDs — the opposite of what is needed. The startup sync below
takes over its role.

**Startup sync — reversed direction.** Unlike the M4b job which synced CRD→PostgreSQL,
the startup sync in M4c runs in the opposite direction: PostgreSQL→CRD. Before the HTTP
server starts, each pod reads all quota and allocation records from PostgreSQL and
upserts the corresponding CRDs, repairing any stale entries left by failed writes. The
pod only registers as ready after this completes. This ensures CRDs are current whenever
a pod is healthy, without the risk of the sync overwriting correct PostgreSQL data with
stale CRD values.

**Rollback analysis.** The following table covers the rollback scenarios:

| Scenario | PostgreSQL | CRD after write | CRD after next pod start | Rollback before restart | Rollback after restart |
|---|---|---|---|---|---|
| Both writes succeed | current | current | current | ✅ clean | ✅ clean |
| PG write fails | unchanged | unchanged | unchanged | ✅ clean | ✅ clean |
| PG succeeds, CRD write fails (retries exhausted) | current | stale | repaired | ❌ stale | ✅ clean |
| PG unavailable | errors | unchanged | unchanged | ✅ clean | ✅ clean |

The problematic case — PostgreSQL succeeds, CRD write fails, rollback before the next
pod restart — requires a non-transient CRD write failure and an operator-initiated
rollback before any pod has restarted. In this case:

1. ERROR log entries identify the affected IDs.
2. `unikorn-quota sync --db-to-crd` can be run manually to repair CRDs before rolling
   back.
3. The operator can also inspect and patch the affected CRDs by hand using the logged
   IDs.

**Consistency controller.** A read-only background controller runs within the identity
process and periodically compares PostgreSQL state against CRD state. It exposes discrepancies
as a Prometheus metric:

```
unikorn_quota_crd_discrepancies{kind="quota|allocation"}
```

and logs the differing IDs at WARN level. It does not attempt to repair anything —
intervention is deliberate and manual. This metric is the primary signal for deciding
when M5 is safe to execute, and it surfaces any CRD write failures that survived retries
and pod restarts. The controller fits naturally as a controller-runtime reconciler alongside the existing
controllers in this repository, and moves to `uni-quota` at M6.

**K8s API access.** Even with the PostgreSQL backend active, `unikorn-quota` retains
its Kubernetes client: `QuotaMetadata` CRDs are still read from the API for display
metadata (display names, descriptions, defaults), and the consistency controller
requires list/get access to `Quota` and `Allocation` CRDs.

Rollback: set `--db-url` to empty or flip `--quota-backend=kubernetes`. CRDs are
current in all but the exceptional case described above; if ERROR log entries or
non-zero discrepancy metrics are present, run `unikorn-quota sync --db-to-crd` first.

### Step M5: Drop CRD writes

Once the team is confident in the PostgreSQL backend, the dual-write to CRDs is removed.
The quota listener reads and writes PostgreSQL exclusively.

**Confidence criteria.** Before taking this step:

- The `unikorn_quota_crd_discrepancies` metric has been zero in production continuously
  over a meaningful period (e.g. several releases or a defined number of days).
- No rollbacks from M4c have been required.
- The `unikorn-quota sync --db-to-crd` tool has been exercised and verified to work
  correctly (it is the break-glass rollback path after M5).

**Code changes.**

- Remove the CRD write path from the allocation and quota handlers.
- Remove the PostgreSQL→CRD startup sync from `main.go`.
- `QuotaMetadata` CRDs are still read for display metadata, so the Kubernetes client
  is retained unless `QuotaMetadata` is separately migrated to static config or a DB
  table (out of scope here).

**CRD type and manifest removal.** The `Quota` and `Allocation` CRD definitions
(`identity.unikorn-cloud.org_quotas.yaml`,
`identity.unikorn-cloud.org_allocations.yaml`) can be removed from the Helm chart in
this step or deferred to a follow-up. Removing them also removes the corresponding Go
types from `pkg/apis/unikorn/v1alpha1/types.go` and their generated deep-copy code.
Deferring is safer — the CRDs are harmless empty resources at this point.

**`unikorn-quota sync --db-to-crd` is retained.** Even after M5, this tool must remain
available. It is the only recovery path if a rollback is ever needed, and it can be used
to reconstruct CRD state for diagnostic purposes.

**Rollback.** Run `unikorn-quota sync --db-to-crd` to populate CRDs from PostgreSQL,
then flip `--quota-backend=kubernetes`. This is a deliberate manual operation; it
should not be needed in normal circumstances and requires operator judgement about
whether the PostgreSQL data is trustworthy before it is written back to CRDs.

### Step M6: Extract quota service to uni-quota

Move all quota code from `uni-identity` into `uni-quota`, replacing the forward-compat
shims with real implementations. No consumer impact — imports already reference
`uni-quota`.

**Code moves.**

- `pkg/quota/openapi/` → `uni-quota/pkg/openapi/`: generated types and client replace
  the shims.
- `pkg/client/allocations.go` → `uni-quota/pkg/client/`: the wrapper replaces the shim.
- `pkg/quota/handler/` → `uni-quota/pkg/handler/`.
- `pkg/handler/quotas/` and `pkg/handler/allocations/` → corresponding packages in
  `uni-quota`.

**RBAC refactor.** The quota handler currently uses identity's in-process `rbac`
package. In `uni-quota` it must use the HTTP-based token validation and scope
enforcement pattern used by region, compute, and other UNI services. `uni-quota`
retains its dependency on `uni-identity` for `principal` and other shared utilities;
the dependency direction is unchanged.

**Binary extraction.** `cmd/unikorn-quota/` moves from `uni-identity` to `uni-quota`
and gains the HTTP listener alongside its existing migration and sync subcommands. The
second HTTP listener is removed from identity's `main.go`. The Kubernetes Service
previously pointing at identity's quota port is updated to point at the new
`unikorn-quota` deployment.

Rollback: redeploy identity with the second listener restored and revert `uni-quota` to
the shim version.

---

## PR Timeline

The table below maps each migration step to individual PRs, human actions, and
deployment gates. PRs within a step can be reviewed in parallel but must be
deployed in the order shown.

| # | Repo | Type | Description |
|---|------|------|-------------|
| 1 | uni-identity | PR | Narrow `pkg/client/allocations.go` to a 3-method `allocationAPI` interface. Pure refactor; existing `openapi.ClientWithResponsesInterface` satisfies it so all consumers compile unchanged. |
| 2 | uni-identity | PR | Add `pkg/quota/openapi/server.spec.yaml` with the 6 quota/allocation endpoints and commit generated code. Try `import-mapping` to reference types from `pkg/openapi`; duplicate definitions if the tool does not support it. No behaviour change. |
| 3 | uni-identity | PR | Create `pkg/quota/handler/` implementing the quota `ServerInterface`. Refactor identity's `Handler` quota/allocation methods to delegate to it — `allocationMutex` and `directclient` move into the quota handler. Add the second HTTP listener in `main.go` and the Kubernetes `Service` manifest for the quota port. Both listeners share the same `pkg/quota/handler/` instance. |
| — | — | **Human** | Create the `uni-quota` repository. |
| 4 | uni-quota | PR | Add forward-compat shims: `pkg/openapi` re-exports quota types from `uni-identity/pkg/quota/openapi` via type aliases; `pkg/client` re-exports `NewAllocations` and `Allocations`; `pkg/openapi/builder.go` for constructing a quota `BaseClient`. Tag an initial release. |
| — | — | **Deploy** | Roll out identity (PR 3). The new quota listener is live but no consumer points at it yet. |
| 5 | uni-kubernetes | PR | Add `--quota-endpoint` flag, `uni-quota` dependency, switch module imports, add dedicated `quota allocationAPI` field to the handler, move Pact contract tests to `consumer/quota/`. |
| 6 | uni-compute | PR | Same five changes as PR 5. |
| 7 | uni-region | PR | Same five changes as PR 5. |
| — | — | **Deploy** | Roll out each consumer as its PR merges. Order does not matter — during the transition both the identity quota endpoint and the new quota listener serve from the same CRDs. |
| 8 | uni-identity | PR | Remove the 6 quota/allocation endpoints and their types from `pkg/openapi/server.spec.yaml`, regenerate. Remove delegation methods from `handler.go` and `handler_allocations.go`. Update `pkg/client/allocations.go` type references from `identityapi` to `quotaapi`. `pkg/handler/quotas/`, `pkg/handler/allocations/`, and `pkg/quota/handler/` are untouched. |
| — | — | **Deploy** | Roll out identity. The quota listener keeps running; consumers are unaffected. |
| 9 | uni-identity | PR | Add `cmd/unikorn-quota/` with schema migration tooling (e.g. `golang-migrate`) and the initial SQL. Add a Helm pre-upgrade Job (`hook-weight: -10`). No reads or writes to the new tables yet. |
| — | — | **Deploy** | Upgrade creates the empty tables. Safe to leave for several releases before the next step. |
| 10 | uni-identity | PR | Add the CRD→PostgreSQL bulk-upsert subcommand to `cmd/unikorn-quota/`. Add a second Helm pre-upgrade Job (`hook-weight: -5`). Column mapping: `id` from `allocation.Name`; `organization_id` / `project_id` from their CRD labels; `resource_kind` / `resource_id` from `ReferencedResourceKindLabel` / `ReferencedResourceIDLabel` labels; `creator_id` from the `CreatorAnnotation` annotation (`Userinfo.Sub`); `creator_principal` from the `CreatorPrincipalAnnotation` annotation (`Principal.Actor`); `tags` from `spec.tags` serialised to JSONB. |
| — | — | **Deploy** | Upgrade runs both Jobs; PostgreSQL is populated. Service remains CRD-backed. Leave for at least one release cycle to build confidence in the migration output before M4c. |
| 11 | uni-identity | PR | Implement the PostgreSQL read/write backend, selected when `--db-url` is set. Dual-write (PostgreSQL first, then CRD with 3-attempt retry and ERROR log on exhaustion). Startup sync in the reversed direction (PostgreSQL→CRD) before the HTTP server registers ready. Retire the M4b Helm Job. Add the consistency controller exposing `unikorn_quota_crd_discrepancies`. Decide whether this controller runs as a background goroutine in the quota listener process or as a separate `cmd/unikorn-quota-controller/` binary following the existing convention. |
| — | — | **Deploy** | Set `--db-url`. Monitor `unikorn_quota_crd_discrepancies` — should reach zero after startup sync completes and stay there. |
| — | — | **Human** | Confirm `unikorn_quota_crd_discrepancies` has been zero continuously over a meaningful period, no M4c rollbacks have been required, and `unikorn-quota sync --db-to-crd` has been tested end-to-end. |
| 12 | uni-identity | PR | Remove the CRD write path from allocation and quota handlers. Remove the PostgreSQL→CRD startup sync from `main.go`. Retain `unikorn-quota sync --db-to-crd` as the break-glass rollback tool. CRD type definitions can stay in the Helm chart for now as a deferred cleanup. |
| — | — | **Deploy** | Point of no return. Rollback now requires running `unikorn-quota sync --db-to-crd` then flipping the backend flag. |
| 13 | uni-quota | PR | Move `pkg/quota/openapi/`, `pkg/client/allocations.go`, `pkg/quota/handler/`, `pkg/handler/quotas/`, `pkg/handler/allocations/`, and `cmd/unikorn-quota/` from identity. Replace all shims with real implementations. Apply the RBAC refactor (HTTP token validation pattern). Add the HTTP listener to the `unikorn-quota` binary. Tag a release. |
| 14 | uni-identity | PR | Remove the now-moved packages and the second listener from `main.go`. Update the Kubernetes Service to point at the `unikorn-quota` Deployment. |
| — | — | **Deploy** | Roll out `unikorn-quota` as a new Deployment, then roll out identity (PR 14). No consumer impact. |
| — | — | **Human** | Optional follow-up: remove `Quota` and `Allocation` CRD definitions from the Helm chart and `pkg/apis/unikorn/v1alpha1/types.go` once no instances remain in any cluster. |
