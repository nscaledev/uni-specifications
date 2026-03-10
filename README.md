# Specifications

## Platform Specification

The **[Unified Architectural Specification](SPECIFICATION.md)** is the single source of truth for the Nscale Cloud Platform. It is the normative reference for all engineers and AI automation agents. Any code or design that does not conform to it is a defect.

Each service repository contains a `CLAUDE.md` that references this document. AI agents should fetch the raw content of `SPECIFICATION.md` directly for full platform context.

## Provider-Specific Specifications

Provider-specific configuration that sits below the platform abstraction boundary is documented separately:

### OpenStack

* [Flavor and Image Handling](specifications/providers/openstack/flavors_and_images.md)
* [External Networks](specifications/providers/openstack/external-networks.md)
