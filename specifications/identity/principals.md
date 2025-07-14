# Identity Principals and Proxies

## Changelog

- v1.0.0 2025-07-14 (@spjmurray): Initial Draft

## Requirements

The goal here is to have a resource provisioned on behalf of another user, but with the following properties:

* The user does not have access to the resource
  * Elevated privileges may have granted to the resource for it to operate
  * The resource is part of a managed service so end user interaction is not desired
* The end user's organization/project has quota allocations deducted from it, not the provisioning service
* The end user is billed for resources, not the provisioning service

## Securing User Access

A service will need to provision resources on behalf of the user it its own organization.
As a result the service can provision end-user accessible interfaces but not have the resource available in the end user's organization.

For example, when running a 3rd party service on managed infrastructure, the user would be presented with either a secure UI offered by that service, or some other secure CLI APIs for running workloads.
At no point should the end user be presented with a key to access the managed infrastructure directly as this may lead to privilege escalation, or interference with the managed service.

## Principals and Proxies

We propose two new terms to encapsulate these concepts.

`Principals` identify the principal end-user responsible for creation and management of the resource.
The principal is ultimately responsible for quota allocation use and billing.

`Proxies` act on behalf of an end-user to provision resources.
A proxy _may_ choose to provision the resource directly in the principal's tenancy, for example the region service doesn't expose any low level resources to end users, so can do this safely.
A proxy may alternatively choose to provision the resource in its own tenancy, for example when provisioning a compute cluster to run on and does not wish to have the end-user gain access via SSH.

### Resource Tenancy Information

Most resources in the system are associated with the user or service that directly provisioned them.
The meaning of these do not change and operate exactly as before.

Principals are an extension that identify the initial originator of a resource.

### Principals

Upon initial entry into the system, i.e. a publicly accessible API, the principal can be generated from:

* Access token introspection, to acquire the actor for audit logs.
* HTTP path introspection, to derive the organization and project the request is for.
  This path introspection information is required for attribution of resource allocations and billing.

The principal must be propagated at every point of the system from then on:

* If the end-user accessible API communicates to another API, it must propagate the principal information.
* If any service creates a resource, the principal information must be persisted in the resource.
  * The principal information in the resource _must_ be enforced to be immutable.
* If a controller communicates with another API, then it must propagate the principal information from the resource the controller is operating on to the remote service.

### Principals for Allocations

Any service that creates, updates or deletes resource allocations must do so using the principal information.

When creating the allocation, the principal can be derived from the request context.

When updating or deleting the allocation, the principal must be derived from the resource the controller is managing the allocation on behalf of.

### Principals for Billing

Billing systems may either poll top level resources and charge for them (top down).
All chargeable resources will either be directly attributable to an end-user tenancy.
Where resources belong to a proxy service, they can be ignored as they are likely to be considered as part of a higher order service already.

They may also poll for actual physical infrastructure in use (bottom up).
For this case, we simply need to propagate the principal information down to resources e.g. virtual machines, as labels that can be polled for actual resource utilization, rather than what was requested.
