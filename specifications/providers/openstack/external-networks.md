# External Networks

One of the challenges that OpenStack users face is the selection of external networks.
Unikorn attempts to &mdash; where possible &mdash; make intelligent decisions for the user to improve UX.

It's entirely possible to have multiple external networks defined that:

* Provision on different L3 subnets.
* Provision differently e.g. directly on the internet as opposed to via a NAT gateway.
* May not actually work as you're expecting e.g. be private.

Question is how do we make that selection?

## Changelog

- v1.0.0 2024-07-23 (@spjmurray): Initial RFC

## Network Metadata

Existing networks contain a bunch of information that is visible to the user, most of it useless.
There is a `is_default` flag, but sadly that is used for QoS rather than a scheduling hint.

Networks do however support arbitrary `tags` that we can use to select resources.

## Network Selection Strategies

### Explicit Selection

Simplest first, allow an operator to specify an explicit network or networks.

How do we make an intelligent decision as to which to use on behalf of the user?
The ordering of networks forms a preferred order of evaluation.

The drawbacks with this approach are that new networks that are added e.g. to facilitate the expansion of address space, are not automatically picked up by the system.
Likewise if an underlying network's ID changes for whatever reason, this will break without modifications.

### Implicit Selection

Rather than specifying a list of network IDs, we can supply a tag, or tags (even a negated match) and use that to select our networks.

It does however place the responsibility on the platform engineers to maintain tags, such as when adding new network prefixes, but it does mean the networks will be picked up on a cache flush without any interaction on Unikorn's part.

### Network Utilization

While it's all well and good having multiple networks, when selecting one, you are left with a choice of use the first, or use a random one.

We can be more intelligent about this by requiring that the Unikorn region controller can probe the network subnet, and determine whether there are enough IP addresses for allocation.
But, at the expense of being too clever, while we could assume we need a NAT gateway address, and an API endpoint or bastion, there is no restriction on what load balancers a user can configure.

Typically you'd have just one, and let HTTP host, or SNI routing do the hard work.
Ideally we'd restrict platform requirements but that requires a fully managed service with no access to Kubernetes etc.

Finally, you have noisy neighbors to worry about, even with a fully managed platform, the addition of an ingress controller (and by extension a floating IP) could be dynamic and on request.
Can we now fulfill that request, or has someone else come along and exhausted all the IP address space?
The only way that would fly is if we reserved IP address space, and that's not cheap, especially with IPv4.

## Specification

At this stage, we probably get most bang for our buck with tag based selection, to implicitly define the set of networks, and a random selection policy.
In future, we can add more intelligent utilization to replace that random decision.

We should also support a single explicit network to provide an easier alternative.
The region definition will look like:

```yaml
apiVersion: regions.unikorn-cloud.org/v1alpha1
kind: Region
spec:
  openstack:
    network:
      externalNetworks:
        # Selector defines what neworks to include in the working set,
        # this is a boolean intersection of all filters defined below.
        selector:
          # Explicit set of network IDs.
          ids:
          - 623bcaf3-7785-496a-a7d9-12919bce5b0f
          # Implict selection of networks that have all these tags.
          tags:
          - unikorn:external-network
```

The tags can be arbitrary, nothing ties them to any code.
