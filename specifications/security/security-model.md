# Unikorn Security Model

## Abstract

RBAC as described in another [specification](../rbac/access-control-lists.md) is the cornerstone of the unikorn security model.
It provides granular access to individual operations on individual endpoints.
These can be aggregated into roles then associated with groups of users granting those users the ability to perform those operations at an applicable API.

What this does not address are higher level abstraction that are more relevant to business logic that this document will address.

## Changelog

- v1.0.0 2024-10-05 (@spjmurray): Initial RFC

## Background

Historically, unikorn was entirely driven by synchronous I iterations from a user agent e.g. a browser. This led to the user of one service requiring permission to use other endpoints on requisite services.
For example, when provisioning a Kubernetes cluster, you also need a cloud identity to be provisioned (providing isolated and constrained credentials), and optionally a physical network (to support baremetal instances) , that will be consumed by Kubernetes cluster provisioners.

The unfortunate knock on effect of this model was that to grant a user permission to provision a cluster also grants them permission to create physical networks.
The inherent danger with is is that VLANs only allow 4094 IDs, and it would be trivially easy to deny service to other users with an errant script.

One solution to this issue is to enforce quotas on resource provisioning.
This can be done either globally, or within the scope of another resource (e.g. an identity can have on physical network), but either of these strategies are complex to implement and manage.

To summarize we have a number of problems:

* Proliferation of permissions that are sensitive and impacting to platform stability.
* Difficulties in quota enforcement.

## Proposal

Taking Kubernetes clusters as an example again, it was desirable for identities and physical networks to be freed automatically when deleted.
Unfortunately at the time, it was impossible to do this asynchronously in a controller.
That would have required a user token to be cached, and that may expire.
A simple event queue was trialed, using Kubernetes events, but this proved hard to maintain, susceptible to injection attacks and unreliable, leading to resource leaks.

It was noted at the time that oauth2 supported X.509 based client authentication.
That allowed a service to authenticate, be granted permissions, and access external APIs asynchronously, thus providing a robust solution to the deletion problem.

By extending this pattern to resource creation at the API level too it has the useful properties of:

* Providing privilege escalation
* Acting on behalf of a user
* Removing the need for a user to have sensitive permissions that could be used in a denial of service attack

Thus a user who needs to provision Kubernetes clusters, only needs that permission.
Permission to create identities and physical networks are delegated to the service itself.

From the perspective of quota management, let's pretend the services themselves work perfectly.
We no longer need to expose lower level resources and enforce quotas at that level, vastly simplifying resource management.
You are now allowed 16 machines in a cluster(s) for example.

All of this leads to a higher level abstraction that is easier to procure for, make deals with, bill for, and ultimately offer a far simpler and less error prone user experience.

## Future Work

* Break down roles further
  * Kubernetes user
  * Compute user
* Higher level abstractions
  * Templated Kubernetes user, this removes more knobs and privileges, with simpler SKUs and billing, you get 8 nodes, that's it.
  * Same for compute etc.

