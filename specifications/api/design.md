# API Design

APIs across Unikorn components all work the same in order to share common functionality for example:

* Authentication and authorization.
* HTTP Request/response logging.
* Audit logging.
* CORS handling.

All new Unikorn components must adhere to these guidelines for consistency.
As Unikorn is modular, and supports 3rd party expansion, we encourage the use of the methodologies defined in this specification.

## Changelog

- v1.0.0 2024-07-17 (@spjmurray): Initial RFC

## OpenAPI

All APIs must use OpenAPI to describe interfaces, giving rise to language agnostic client, server and type generation.
It also means middleware can be written once to the OpenAPI specification and shared across all APIs.

### Paths

API paths e.g. `/api/v1/resource/{resourceID]` are standardized across components.

At the top level, you may have:

* `/oauth2` - implements a specification e.g. oauth 2.
* `/api/v1` - implements APIs specific to the component.

Delving further into API specific paths you may have:

* `/api/v1/resource` - a global resource kind.
* `/api/v1/organizations/{organizationID}/resource` - an organization scoped resource kind.

For global resources, they SHOULD be marked as requiring oauth2 authentication where possible.
For organization scoped resources, you MUST use oauth2 authentication.
See [Authentication](#authentication) for more information.

Furthermore, all non-global resources MUST be scoped to an organization.
See [RBAC](#rbac) for more information.

### Resource Identifiers

Resource identifiers MUST be formatted as UUIDv4, as defined in the [Resource Metadata Specification](./resource-metdata.md).

The resource parameter name MUST be standardized across all components as it may be used as structured metadata in logs, and then used to index entries that correspond to a resource that span multiple components.
Examples include:

* `organizationID` - the organization a resource belongs to.
* `projectID` - the project a resource belongs to.

One example of this use is the [Audit Logging Specification](../logging/audit-logging.md) to provide resource scope in a generic manner.

## Authentication

[Unikorn Identity](https://github.com/unikorn-cloud/identity) is a federated oauth2/OIDC service for user identity, authentication and authorization.
When a user logs in, they are granted an access token.
As per the oauth2 specification, the access token can be used against a resource server to grant access to a resource for the user.
This token is opaque, meaning that only the Identity service can parse and validate it.

To that end, all API that MUST authenticate a resource access must perform a callback to the Identity service &mdash; with the access token &mdash; in order to authenticate the user.
Typically this can take the form of a token introspection call ("userinfo" - to get the subject for audit logging), or a call to get the user's access control list for RBAC.

## RBAC

All endpoints that can support RBAC MUST.

RBAC allows granular access to resources based on the Identity service's concepts of organizations and projects.
Permission to access an endpoint are granted based on the endpoint group, a free form user-defined name, and a requested permission e.g. create, read, update, delete.

Go users MUST use the [reference implementation](https://github.com/unikorn-cloud/identity/blob/main/pkg/rbac/handler.go) as defined by the Identity service.
Other languages should endeavor to copy those semantics as best possible, better still contribute a library.
