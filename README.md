# Unikorn Specifications

![Unikorn Logo](https://raw.githubusercontent.com/unikorn-cloud/assets/main/images/logos/light-on-dark/logo.svg#gh-dark-mode-only)
![Unikorn Logo](https://raw.githubusercontent.com/unikorn-cloud/assets/main/images/logos/dark-on-light/logo.svg#gh-light-mode-only)

## Overview

This is the central location for all specifications that are implemented by Unikorn components.
Large bodies of work should have a specification created and approved before starting any work.
For work that defines an interface for an external 3rd party entity e.g. logging, UX, a specification forms an interface contract and is mandatory.

## Index

Specifications are broken down into conceptual areas as they span multiple components.

### API

Unikorn components are split up into micro-services with RESTful APIs that can be leveraged by Web browsers, CLI tools etc.
API specifications document common API conventions that apply across the platform to foster code reuse.

* [API Design](specifications/api/design.md)
* [API Resource Metadata](specifications/api/resource-metdata.md)

### RBAC

Role based access control [RBAC] provides fine-grain permissions for granting access to resources within the platform.

* [Access Control Lists](specifications/rbac/access-control-lists.md)

### Logging

Logging is more than just for debugging.
Logging is structured data and can be used as an event source for consumption by external instrumentation, for example for audit logging of resource modification and billing.

* [Audit Logging](specifications/logging/audit-logging.md)

### Providers

Unikorn strives to be platform agnostic.
Although it is primarily written with OpenStack in mind, care has been taken to abstract away platform specific details through architectural descisions.
Where platform specific configuration and setup is required, or coordination is required between different teams, this linkage is specified here.

#### OpenStack

* [Flavor and Image Handling](specifications/providers/openstack/flavors_and_images.md)
* [External Networks](specifications/providers/openstack/external-networks.md)
