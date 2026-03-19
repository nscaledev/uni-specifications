# KinD-Based Integration Testing Strategy

## Problem Statement

The Nscale Cloud Platform currently has no automated integration tests that run against a real
deployed service instance. Meaningful end-to-end tests require a live cluster with cert-manager,
ingress-nginx, and a running identity server — making them impractical to run without additional
tooling. This has several consequences:

- Helm chart errors (hardcoded names, missing labels, incorrect RBAC bindings) are not caught until
  manual deployment.
- RBAC correctness is validated only by unit tests mocking the permission logic, not by real HTTP
  calls with real tokens.
- Kubernetes controller reconciliation (organization-controller, project-controller,
  oauth2client-controller) is not exercised in CI.
- The barrier to running tests locally discourages shift-left quality practices.

## Changelog

| Version | Author | Date | Notes |
|---------|--------|------|-------|
| v0.1 | Platform Engineering | 2026-03 | Initial draft |
| v0.2 | Platform Engineering | 2026-03 | Reflect `hack/ci/` layout; go.mod-pinned infra deps; setup-infra/install separation; ingress-nginx via Helm |
| v0.3 | Platform Engineering | 2026-03 | Reflect implemented state: mTLS auth model, two-principal RBAC matrix, protected role constraint, fixtures via OpenAPI client |
| v0.4 | Platform Engineering | 2026-03 | Fixtures as ecosystem primitive; self-contained test suites; full coverage fixture topology (multi-org, multi-project, all role scopes, isolation axes) |

## Design Principles

**1. The cluster is the root of trust.**
A kubeconfig with admin access to a Kubernetes cluster is the only external dependency required
to bootstrap the entire test environment. There are no pre-existing users, tokens, or external
identity providers required.

**2. Fixtures via the HTTP API using mTLS.**
Test prerequisites (Organization, Project, Group, ServiceAccount) are created by calling the
identity HTTP API using an mTLS client certificate. The certificate CN maps directly to an RBAC
role — no bearer token exchange is needed. The system account (`ci-fixtures`) is pre-configured
in the Helm release via `hack/ci/test-values.yaml`, granting it the `platform-administrator` role.

This approach ensures the fixture creation path exercises the same API surface as production. The
mTLS back-channel provides the privileged access needed to bootstrap resources without relying on
any pre-existing bearer token.

**2a. Fixtures are a composable ecosystem primitive.**
`hack/ci/fixtures` is not only used by identity's own test suite — it is the authoritative source
of identity primitives for the entire platform. Downstream services (region, compute, kubernetes)
call it to obtain a working set of organisations, projects, groups, and service account tokens
before running their own tests. Its output contract is therefore as stable and intentional as
`hack/ci/install`.

The fixture topology is deliberately rich (see [Fixture Topology](#fixture-topology)) so that any
consumer can exercise org isolation, project isolation, and every role scope without creating
additional identity resources themselves.

**2b. Each service's test suite is self-contained.**
While `hack/ci/fixtures` provides the shared ecosystem bootstrap, each service's own Ginkgo suite
must not depend on a pre-populated `test/.env` file produced by an external step. The suite
creates everything it needs in `BeforeSuite` and destroys it in `AfterSuite`. The fixture
creation logic lives in a reusable Go package (e.g. `pkg/testing/fixtures`) that both the
`hack/ci/fixtures` CLI and `BeforeSuite` import — one implementation, two consumers, no
duplication.

**3. The composable install and fixtures model.**
Each service defines a single install unit (`hack/ci/install`) that knows how to deploy that
service into a running cluster. All CI-related scripts live under `hack/ci/` in each service
repo. Install units are:
- **Self-contained** — they carry all knowledge needed to install the service
- **Idempotent** — safe to call multiple times; checks before acting
- **Composable** — higher-level services call dependency install units directly

Cluster-level infrastructure prerequisites (cert-manager, ingress-nginx, unikorn-core) are
installed by a separate `hack/ci/setup-infra` script. This is called once per cluster, not
once per service install. Because it is idempotent, higher-level service CIs can call it
without risk of duplication or conflict.

This means install logic is defined once per service and reused by every service that depends on
it. When identity's install procedure changes, all consumers pick it up automatically with no
duplication.

```
# Identity CI:
hack/ci/setup-infra
hack/ci/install   --namespace unikorn-identity-$RAND --release-name identity-$RAND > identity.env
. identity.env
hack/ci/fixtures  --base-url "$IDENTITY_BASE_URL" --namespace "$IDENTITY_NAMESPACE" \
                  --ca-cert "$IDENTITY_CA_CERT" > identity-fixtures.env

# Compute CI (when it needs identity and region as dependencies):
hack/ci/setup-infra                   # idempotent — safe to call at the top of any CI run
../identity/hack/ci/install   --namespace unikorn-identity-$RAND --release-name identity-$RAND \
                               > identity.env
. identity.env
../identity/hack/ci/fixtures  --base-url "$IDENTITY_BASE_URL" --namespace "$IDENTITY_NAMESPACE" \
                               --ca-cert "$IDENTITY_CA_CERT" > identity-fixtures.env
. identity-fixtures.env
../region/hack/ci/install     --namespace unikorn-region-$RAND --release-name region-$RAND \
                               > region.env
. region.env
hack/ci/install               --namespace unikorn-compute-$RAND --release-name compute-$RAND
# compute's own BeforeSuite uses identity-fixtures.env tokens to authenticate
```

**4. go.mod as version source of truth.**
`go.mod` is the authoritative definition of exactly what gets tested — not just for the service
under test, but for every infrastructure dependency too.

When a service is built for testing, it is built from the current source tree using
`make images-kind-load` (KinD) or `make images` (Colima). Any `replace` directives in `go.mod`
(e.g., pointing a dependency at a local branch) are automatically incorporated into the built
image.

The same principle applies to Helm chart dependencies such as `unikorn-core`. Rather than
installing a published chart from a Helm repository (which may be at a different version),
`hack/ci/setup-infra` uses `go list -m -f '{{.Dir}}' github.com/unikorn-cloud/core` to resolve
the exact module directory from the Go module cache (respecting any `replace` directives), copies
it to a temp directory, runs `helm dependency build`, and installs from that local path. This
means the ClusterIssuer configuration is always in sync with the version of core that the service
under test depends on.

**5. Full randomisation.**
Every KinD test run uses randomly generated Helm release names and Kubernetes namespaces. Ingress
hostnames are derived from the release name. All test resource names are UUID-based. This ensures
that hardcoded names in Helm templates, application code, or RBAC rules are caught immediately
rather than being masked by consistent environment configuration.

**6. Identical local and CI environments.**
The same `hack/ci/` scripts and `hack/ci/kind-config.yaml` are used both locally by developers
and by GitHub Actions. There is no CI-specific deployment path — the same commands that a
developer runs locally are the exact commands the workflow executes.

## Infrastructure Stack

The following components must be present in the cluster before a service can be deployed:

```
cluster  (KinD: hack/ci/kind-config.yaml — extraPortMappings for 80/443)
  ├── cert-manager             (from Helm repo; version unpinned — latest)
  ├── ingress-nginx            (from official Helm chart; KinD-specific values applied when IS_KIND)
  └── unikorn-core             (resolved via go module cache at exact commit from go.mod)
       ├── ClusterIssuer: unikorn-issuer        (signs ingress TLS certs)
       └── ClusterIssuer: unikorn-client-issuer (signs mTLS client certs)
```

These are installed by `hack/ci/setup-infra`, which is called as an explicit step before any
service install. Because it is idempotent, higher-level service CIs can call it at the start of
their run without duplication or conflict.

**ingress-nginx** is installed from the official `ingress-nginx/ingress-nginx` Helm chart. When
the cluster is detected as KinD (kubectl context matches `kind-*`), additional values are applied
to enable hostPort mode and NodePort service type, so that KinD's `extraPortMappings` route
host ports 80/443 to the nginx controller. On non-KinD clusters these overrides are omitted and
the chart uses its defaults.

**unikorn-core** creates the `unikorn-issuer` and `unikorn-client-issuer` ClusterIssuers via its
own Helm chart. `setup-infra` resolves the module via `go list -m -f '{{.Dir}}'`, copies it to
a writable temp directory (the module cache is read-only), builds the Helm dependencies, and
installs. The `unikorn-ca` secret's CA certificate is extracted and written to `hack/ci/ca-bundle.pem`
for use by test HTTP clients.

### Hostname and TLS

KinD `extraPortMappings` expose host ports 80 and 443 to ingress-nginx running inside the
cluster. Ingress hostnames use the `nip.io` wildcard DNS service:

```
identity-<suffix>.127.0.0.1.nip.io  →  resolves to 127.0.0.1  →  KinD host ports  →  ingress-nginx
```

This requires no `/etc/hosts` modification and works identically locally and in CI.

TLS certificates are issued by cert-manager via the `unikorn-issuer` ClusterIssuer (created by
unikorn-core). The CA certificate backing that issuer is extracted by `hack/ci/setup-infra` and
written to `hack/ci/ca-bundle.pem`. Test HTTP clients load this file as their trusted CA —
`InsecureSkipVerify` is not used. `hack/ci/ca-bundle.pem` is gitignored; it is regenerated on
each `make kind-infra` run.

### Install timing considerations

After Helm install completes, `hack/ci/install` performs two additional waits before returning:

1. **JOSE signing key** — the identity server creates `signingkey/unikorn-identity-jose`
   asynchronously after acquiring the leader lease. The script polls for up to 60 s.
2. **nginx TLS reload** — cert-manager issuing the ingress certificate does not immediately
   cause nginx to serve it. The script polls `openssl s_client` for up to 120 s until the
   correct SAN appears. This prevents spurious TLS failures in the fixture step.

### Cluster-scoped resource conflicts

Identity's Helm chart creates `ValidatingAdmissionPolicy` resources, which are cluster-scoped and
survive namespace deletion. Installing a second release with a different name while the first
release's cluster-scoped resources still exist causes conflicts. `hack/ci/install` automatically
uninstalls all previous `identity-*` releases (across all namespaces) before deploying.

## Auth Bootstrapping

Identity is itself the OIDC/OAuth2 provider, so there is no external identity provider to
authenticate against during testing. The bootstrap sequence is:

1. A system account (`ci-fixtures`) is pre-configured in the identity Helm release via
   `hack/ci/test-values.yaml` (passed as `--values` to `hack/ci/install`). System accounts
   authenticate via mTLS client certificates; the certificate CN maps directly to an RBAC role.
2. `hack/ci/fixtures` (or `BeforeSuite` via the shared fixtures package) issues a short-lived
   mTLS client certificate for `ci-fixtures` from the `unikorn-client-issuer` ClusterIssuer
   using cert-manager's `Certificate` CRD, via `controller-runtime` (no `kubectl` shell-outs).
3. It calls the identity HTTP API using the generated `openapi.ClientWithResponses` client over
   mTLS. Every request carries an `X-Principal` header (base64url-encoded JSON of the principal
   struct) as required by the middleware.
4. It creates the full fixture topology (see below): two organisations, each with two projects,
   each with groups covering all role scopes, plus an unaffiliated service account.
5. When invoked as `hack/ci/fixtures`, the resulting bearer tokens and resource IDs are written
   as a `.env` fragment to stdout and sourced by downstream CI. When invoked from `BeforeSuite`,
   they are stored in suite-level variables — no file is written.
6. All HTTP API test calls use these bearer tokens.

### mTLS authentication model

The mTLS and bearer-token authentication mechanisms are mutually exclusive:

- **mTLS (system accounts):** the certificate CN maps directly to an RBAC role defined in the
  Helm values. No token exchange is needed. Used by `hack/ci/fixtures` for bootstrapping.
- **Bearer token (service accounts / users):** used by the e2e test suite for all API calls.

An `X-Principal` header (base64url-encoded JSON `{"actor":"<cn>"}`) must accompany every mTLS
API request. The middleware uses this header for principal propagation.

## Protected Roles

Roles marked `protected: true` in the identity Helm values are **not visible or assignable via
the public API**. They may only be granted via Helm values at deployment time:

```yaml
# hack/ci/test-values.yaml
systemAccounts:
  ci-fixtures: platform-administrator
```

The public non-protected roles (assignable via the API) are:

| Role | Scope | Identity permissions |
|------|-------|---------------------|
| `administrator` | Organization | Full CRUD on all identity resources |
| `auditor` | Organization | Read-only on all identity resources |
| `user` | Project | `identity:projects: [read]` at project scope; limited org-level access |
| `reader` | Project | Read-only at project scope |

`hack/ci/fixtures` assigns `administrator` (not `platform-administrator`) to the admin test group,
and `user` to the user test group. The `platform-administrator` role is granted to `ci-fixtures`
via Helm values and is used only for the mTLS bootstrap step.

## Fixture Topology

The fixture set is designed to cover every meaningful scoping and isolation dimension in a single
pass. Downstream consumers can rely on these resources existing and use the exported tokens
directly without creating additional identity primitives.

```
Organisation 1  (TEST_ORG1_ID)
  ├── Project 1  (TEST_PROJECT1_ID)
  │    ├── admin-group    → administrator role  → ci-admin-sa    → ADMIN_TOKEN
  │    ├── user-group     → user role           → ci-user-sa     → USER_TOKEN
  │    ├── reader-group   → reader role         → ci-reader-sa   → READER_TOKEN
  │    └── auditor-group  → auditor role        → ci-auditor-sa  → AUDITOR_TOKEN
  └── Project 2  (TEST_PROJECT2_ID)
       └── user-group     → user role           → ci-p2-user-sa  → PROJECT2_USER_TOKEN

Organisation 2  (TEST_ORG2_ID)
  └── Project 1  (TEST_ORG2_PROJECT1_ID)
       └── admin-group    → administrator role  → ci-org2-admin-sa → ORG2_ADMIN_TOKEN

(no organisation)
  └── ci-unaffiliated-sa  (no group membership) → UNAFFILIATED_TOKEN
```

### What each principal tests

| Token | Scope | Tests |
|-------|-------|-------|
| `ADMIN_TOKEN` | Org 1, org-scoped | Full CRUD on all org 1 identity resources; the workhorse for downstream test setup |
| `USER_TOKEN` | Org 1, Project 1 | Project-scoped access; 403 on org-admin endpoints |
| `READER_TOKEN` | Org 1, Project 1 | Read-only at project scope; 403 on any write |
| `AUDITOR_TOKEN` | Org 1, org-scoped read-only | Read-only across org; 403 on any write |
| `PROJECT2_USER_TOKEN` | Org 1, Project 2 | Project isolation — cannot see Project 1 resources |
| `ORG2_ADMIN_TOKEN` | Org 2, org-scoped | Org isolation — cannot see Org 1 resources |
| `UNAFFILIATED_TOKEN` | None | Hard denial — 403 on every authenticated endpoint |

### Coverage axes

- **Permission level**: administrator, auditor, user, reader — all four built-in scopes
- **Org isolation**: `ORG2_ADMIN_TOKEN` must not see Org 1's resources and vice versa
- **Project isolation**: `PROJECT2_USER_TOKEN` must not see Project 1's resources
- **Hard denial**: `UNAFFILIATED_TOKEN` is authenticated but has no permissions anywhere
- **Data-level filtering**: `USER_TOKEN` on `GET /serviceaccounts` returns 200 but only its own
  record — tests must distinguish endpoint-level denial (403) from data-level filtering (200 with
  reduced result set)

### Downstream role layering

Downstream services define their own roles (e.g. `kubernetes:clusters:read`). These are assigned
to the groups already present in the fixture topology — no new groups or service accounts need to
be created. The administrator group in Project 1 is the natural target for downstream admin roles;
the user group for standard access.

## RBAC Matrix Testing

The RBAC permission matrix is tested systematically at the HTTP level. Tests are structured as
Ginkgo `DescribeTable` entries with one row per endpoint, covering all principals from the fixture
topology that are relevant to the endpoint's access control.

The endpoint list is currently hand-maintained in `test/e2e/rbac_matrix_test.go`. When a new
endpoint is added, a corresponding entry must be added to the relevant table(s).

### Data-level vs endpoint-level access control

Some endpoints return 200 for a principal that has limited access, but filter the response to
only the records that principal is permitted to see. The `GET /serviceaccounts` endpoint is an
example: a service account with the `user` role receives 200 but only sees its own record.
Tests must distinguish between endpoint-level denial (403) and data-level filtering (200 with
reduced result set).

## Developer Workflow

```sh
# One-time local setup — KinD:
make kind-cluster kind-infra

# One-time local setup — Colima (Mac):
colima start --kubernetes && make kind-infra

# Deploy and test (random release name/namespace; suite is self-contained):
make kind-install test-e2e-ci

# Or all at once (KinD only):
make kind-test

# Re-run tests without redeploying:
make test-e2e

# Focus on a specific test during development:
make test-e2e-focus FOCUS="administrator"

# Reuse an existing deployment (pin the suffix):
KIND_SUFFIX=abc12345 make test-e2e-ci

# Run fixtures standalone (for downstream service CI):
. test/.env.install
go run ./hack/ci/fixtures/... \
  --base-url "$IDENTITY_BASE_URL" \
  --namespace "$IDENTITY_NAMESPACE" \
  --ca-cert "$IDENTITY_CA_CERT" \
  > identity-fixtures.env
```

The default KinD cluster name is `identity-test` (not `kind`) to avoid interfering with a
developer's existing default KinD cluster. The cluster name is controlled by `KIND_CLUSTER` and
can be overridden on the command line.

See `docs/integration-testing.md` in the identity repository for the full local developer guide,
including prerequisites, troubleshooting steps, and the pre-PR checklist.

## Open Questions

- Should `hack/ci/setup-infra` pin cert-manager and ingress-nginx versions, or always use latest?
  (unikorn-core is already pinned via go.mod; cert-manager and ingress-nginx are unpinned today.)
- Should the RBAC expected-outcomes table be driven from the OpenAPI spec automatically, so that
  new endpoints cause test failures rather than silent omissions?
- Should downstream services assign their roles to the shared fixture groups, or create their own
  groups within the fixture orgs/projects? The former keeps the token set stable; the latter gives
  each service full isolation of its own role assignments.
