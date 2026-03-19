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
| v0.4 | Platform Engineering | 2026-03 | Align with implemented state: single-org/single-project fixture topology (admin + user principals); Colima as first-class alternative to KinD; correct developer workflow |

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

**3. The composable install model.**
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
hack/ci/install --namespace unikorn-identity-$RAND --release-name identity-$RAND

# Compute CI (when it needs identity and region as dependencies):
hack/ci/setup-infra                   # idempotent — safe to call at the top of any CI run
../identity/hack/ci/install --namespace unikorn-identity-$RAND --release-name identity-$RAND
../region/hack/ci/install   --namespace unikorn-region-$RAND   --release-name region-$RAND
hack/ci/install             --namespace unikorn-compute-$RAND  --release-name compute-$RAND
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

### Cluster type support

`hack/ci/setup-infra` and `hack/ci/install` both detect the current kubectl context:

| Context prefix | Cluster type | Image loading |
|----------------|-------------|---------------|
| `kind-*` | KinD | `make images-kind-load` (build + `kind load`) |
| `colima` / `colima-*` | Colima | `make images` (build only; images are immediately available in the VM) |
| anything else | Remote/cloud | No build; images pulled from registry |

KinD-specific ingress-nginx values (hostPort mode, NodePort service type, control-plane
tolerations) are applied only when the context is `kind-*`. On Colima or cloud clusters the
chart uses its defaults, relying on the cluster's own ingress or load-balancer provisioner.

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
2. `hack/ci/fixtures` issues a short-lived mTLS client certificate for `ci-fixtures` from the
   `unikorn-client-issuer` ClusterIssuer using cert-manager's `Certificate` CRD, via
   `controller-runtime` (no `kubectl` shell-outs).
3. It calls the identity HTTP API using the generated `openapi.ClientWithResponses` client over
   mTLS. Every request carries an `X-Principal` header (base64url-encoded JSON of the principal
   struct) as required by the middleware.
4. It creates: `Organization` → waits for org-controller to provision the backing namespace →
   lists roles → creates two `Group` resources → creates a `Project` → creates two
   `ServiceAccount` resources.
5. The resulting bearer tokens and resource IDs are written to `test/.env` and loaded by the
   Ginkgo test suite.
6. All HTTP API test calls in the e2e suite use these bearer tokens.

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

## RBAC Matrix Testing

The RBAC permission matrix is tested systematically at the HTTP level using two principals:

| Principal | Role | Token variable |
|-----------|------|---------------|
| `ci-admin-sa` | `administrator` (org-scoped) | `ADMIN_AUTH_TOKEN` |
| `ci-user-sa` | `user` (project-scoped) | `USER_AUTH_TOKEN` |

Tests are structured as Ginkgo `DescribeTable` entries:

- **`administrator` table** — expects 200 on all org-level identity read endpoints
- **`user` denied table** — expects 403 on org-admin endpoints the user role has no access to
- **`user` self-view table** — expects 200 on service account list (server filters to own record)

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
# One-time local setup (creates cluster, installs infra):
make kind-cluster kind-infra

# Deploy and test (uses random release name/namespace each time):
make kind-install kind-fixtures test-e2e-ci

# Or all at once:
make kind-test

# Re-run tests without redeploying:
make test-e2e

# Focus on a specific test during development:
make test-e2e-focus FOCUS="administrator"

# Reuse an existing deployment (re-run fixtures against the last install):
make kind-fixtures test-e2e-ci
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
