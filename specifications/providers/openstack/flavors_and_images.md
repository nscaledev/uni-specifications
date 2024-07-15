# Flavor and Image Handling

This document addresses a number of concerns around the handling of flavors and images for the region controller and other services that consume its API.

We do not intend to constrain operators (or indeed a business) in any way, as such we should approach everything in an abstract manner.
One manifestation of this could be a heterogeneous region comprising of hardware (specifically GPUs) from different vendors, within that different models or generations, and different hardware typologies.

For example, we may have a class of hardware with NVIDIA A100s, another with H100s, then a different class of hardware with 2 AMD Mi250s, and another with 4 Mi250s, etc.

While this problem applies to CPUs too, that's readily supported by OpenStack for example.

## Changelog

- v1.0.0 2024-06-26 (@spjmurray): Initial RFC

## Applying Images to Hardware

Images are, by necessity from a performance standpoint, pre-baked to include any components that would by other means need to be downloaded or compiled.
This includes, but is not limited to, Kubernetes container images and GPU drivers.

Images are expected to export metadata that ensures:

* It is only run with the corresponding Kubernetes version asked for by the user to provide consistent start up performance, reducing end user cost and crucially preventing support queries when something doesn't perform as expected.
* It is only run on a class of hardware that is able to use it, running an image with an NVIDIA driver on AMD hardware is not going to work.
  * Further more, if a driver is specific to a certain model of GPU, then that needs to be taken into account also.

## Scheduling Constraints

Kubernetes autoscaling is another consideration.
When pods are scheduled, they may have a resource requirement that needs to be met, specifically a GPU allocation.
Cluster Autoscaler is unable to derive what hardware is available on a node without a node in the first instance e.g. scaling from zero.
As such we need to annotate machine deployments with a descriptor that communicates the GPU type (which is arbitrary, but should be based on the vendor and model as that's unique for a hardware class), and the number of GPUs.
When a pod comes along requesting 4 AMD GPUs, then the autoscaler can fulfill this request by instantiating the number of nodes required to fulfill this request.

## UX Considerations

While it is somewhat fair to make the assumption that a UI could separately store a mapping from flavor to metadata, it also leads to an unmanageable sprawl of data requiring synchronization across organizational units.

If we consider this from the perspective of CLI tooling, then we cannot hard code this mapping, as it will go out of date, and adding an API to extract this information from a centralized location is just adding more work to what is ostensibly a conceptually simple problem.

## Flavor and Image Metadata

Flavors are assumed to be immutable, at the very least, we would need the following information.

* Baremetal or virtual
* GPU vendor
* GPU model
* GPU count (may be either virtual or physical)

Images also need to communicate the following, these are considered mutable e.g. can have constraints reduced if required and possible:

* Kubernetes version (if applicable)
* Baremetal or virtual, or both!
* GPU vendor
* GPU models (driver may be compatible with multiple models)
* GPU physical memory
* GPU driver version (some people may care about this, ideally we should strive to keep people updated as part of a service so this is never a problem, merely a prompt that we need to roll out an upgrade if required).

Images are, at present, automatically selected by the system, the algorithm would look like:

* Find all images that fulfill the optional Kubernetes requirement
* Find all images that fulfill the virtual or baremetal requirement
* Find all images that fulfill the GPU vendor requirement
* Find all images that fulfill the GPU model requirement
* Select the most recent, as that will have fewest CVEs as part of our security guarantee

### Flavor Metadata Mapping

In the old days, we were using a homogeneous platform, and only the number of GPUs was relevant.
It was sufficient to do a regular expression match on flavor properties and extract the number of GPUs from the key in lieu of the `resource:VGPU` or `resource:PGPU` properties not being set.

This forces the underlying platform team to conform to a specific property format that is undesirable, and most likely will make the beardies irate.

This specification opts for a less intrusive and more flexible approach.

A typical region deployment may look like:

```yaml
apiVersion: region.unikorn-cloud.org/v1alpha1
kind: Region
metadata:
  id: dbb911ef-c336-4bfd-917e-9ebc1fadea5e
  labels:
    unikorn-cloud.org/name: my-region
spec:
  provider: openstack
  openstack:
    compute:
      flavors:
        policy: ExcludeAll
        include:
        - id: 43bacb02-eecc-481c-bdc2-4d07f50f9cc1
        - id: d6c65669-d141-4e8b-ad2e-b4641c86e5fa
        - id: f81a2c71-32ab-4f07-a9bf-f4bd5ed97025
          baremetal: true
          gpu:
            vendor: AMD
            model: Mi250
            memory: 192Gi
            count: 2
```

With this approach, inclusion in the flavor list is strictly opt-in.
Flavors are indexed by ID, which is immutable to prevent unnecessary breakage.

Most information about a flavor is generic, e.g. CPU count, memory and disk.
Where we may want to expend this is for generic compute resource which would advertise the CPU vendor and generation, in the case a specific hardware feature is required.

GPUs should be advertised as they would logically appear to a end users, or indeed as required by Kubernetes autoscaler scheduling, irrespective of how many physical cards there are.
Memory should be advertised as per logical GPU.

Where a flavor is baremetal, we can indicate this specifically with a flag, rather than trying to derive this from the underlying region.
Likewise with any GPU configuration.
Thus we now have full control over what information is available and how it's presented.

> [!NOTE]
> We've shown the least permissive option here, we can easily have an `IncludeAll` policy. where the `include` merely augments flavor metadata, and an `exclude` can drop flavors that shouldn't be exposed, beyond the usual built-in rules.

### Image Properties

Images are a lot easier to handle, as they allow the definition of mutable metadata.

As defined by this specification we expect:

```json
{
  "properties": {
    "unikorn:gpu_vendor": "AMD",
    "unikorn:gpu_models": "Mi250,Mi300",
    "unikorn:gpu_driver_version": "v1.2.3",
    "unikorn:kubernetes_version": "v1.2.3",
    "unikorn:virtualization": "any",
    "unikorn:digest": "ZnViYXI="
  }
}
```

The GPU fields are optional, the existence of `gpu_vendor` defines this as compatible with a GPU flavor, and the `gpu_models` and `gpu_driver_version` are required.
The `gpu_models` is formatted as a CSV, and must correspond exactly to values defined by flavor mapping.

The Kubernetes fields are optional and only defined when intended to be used for deployment of Kubernetes services.

The virtualization fields define where the image can be used, `any` means it can be used anywhere, `baremetal` and `virtualized` indicate it can be only used on flavors that match that usage.

The digest is a digital signature that may (but is highly recommended to) be applied to images to aid in image visibility and integrity.

> [!NOTE]
> At present this is the base64 encoded SHA256 hash of the image ID, but we may in future extend this to include a hash over the metadata as well to detect any tampering that may cause platform failure.
> For example `SHA256(ID | key=value[,key=value]*)` where keys are lexically ordered.

