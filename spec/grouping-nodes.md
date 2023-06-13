# Grouping nodes

## Problem description

Today a DataPlaneRole CR defines a grouping of OpenStackDataPlaneNode CRs to
make it possible to define configuration once but apply it to each Node having
the same Role. Today a DataPlaneNode can only belong to a single Role. Today
there is CRD field level inheritance defined between Role and Node. If a CRD
field is defined both in the Role and the Node then the value from the Node
will take precedence. With today's model the DataPlaneRole has a direct
grouping effect for the NovaExternalCompute CRs as well.

However if we move the EDPM related OpenStack service creation from the
dataplane-operator to the respective service-operator [as proposed in a
separate spec](https://github.com/openstack-k8s-operators/docs/pull/41), then
DataPlaneRole based grouping will not be automatically applied. So we need a
different solution.

Moreover the current restriction that a DataPlaneNode can only belong to a
single DataPlaneRole is limiting. From nova perspective a set of compute nodes
can be configured with CPU pinning while from neutron perspective a set of
compute nodes can be configured with DPDK. If the two sets of nodes overlap but
are not equal then there will be a need for additional roles to be defined for
each combination of the configuration (pinning with DPDK, pinning without
DPDK, sharing with DPDK, sharing without DPDK). This shows that the current
restriction can lead to the explosion of the number of necessary groups to
model the different combinations of service configs.

## Proposed change

Allow an EDPM node to belong to multiple groups by defining OpenStack service
specific profiles. The DataPlaneRole could remain as is today but it would
represent the grouping of the EDPM nodes only just from the generic
hardware, host OS, infrastructure configuration perspective.

Each service operator that has EDPM specific service CRD (e.g.
NovaExternalCompute, OVNMetadataAgent) can define a Profile CRD (e.g.
ComputeProfile, NetworkProfile) to describe the service specific configuration
of a set of EDPM nodes. Then the human deployer can use annotations on the
DataPlaneRole or DataPlaneNode to select a single Profile per service type.

### Example

ComputeProfile with shared CPUs
```yaml
apiVersion: nova.openstack.org/v1beta1
kind: ComputeProfile
metadata:
    name: dell-r740-shared-cpus
spec:
    customComputeServiceConfig: |
    [compute]
    cpu_shared_set = 4-12,^8,15
    [DEFAULT]
    cpu_allocation_ratio = 4.0
```

ComputeProfile with pinned CPUs
```yaml
apiVersion: nova.openstack.org/v1beta1
kind: ComputeProfile
metadata:
name: dell-r740-dedicated-cpus
spec:
    customComputeServiceConfig: |
    [compute]
    cpu_dedicated_set = 4-12,^8,15
    [DEFAULT]
    cpu_allocation_ratio = 1.0
```

OpenstackDataPlaneRole/edpm-compute
```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneRole
metadata:
    name: edpm-compute
    annotations:
        nova.openstack.org/cell: nova-cell1
        nova.openstack.org/compute-profile: dell-r740-shared-cpus
        telemetry.openstack.org/telemetry-profile: collector
spec:
...
```
This Role defines a set of EDPM compute nodes with shared CPUs.


OpenstackDataPlaneNode/edpm-compute-0
```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNode
metadata:
    name: edpm-compute-0
spec:
    role: edpm-compute
...
```
This EDPM node is a compute that has shared CPUs as the Role selected the
dell-r740-shared-cpus ComputeProfile

OpenstackDataPlaneNode/edpm-compute-1
```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNode
metadata:
    name: edpm-compute-1
    annotations:
        nova.openstack.org/compute-profile: dell-r740-dedicated-cpus
        neutron.openstack.org/network-profile: dpdk
spec:
    role: edpm-compute
...
```
This EDPM node is a compute with pinned CPUs and DPDK data path as the
compute-profile annotation on the Node overrides the same annotation from the
Role. But this EDPM node still has the a collector telemetry-profile inherited
from the Role.

NovaExternalCompute=edpm-compute-0
```yaml
apiVersion: nova.openstack.org/v1beta1
kind: NovaExternalCompute
metadata:
    name: edpm-compute-0
spec:
    dataplaneNodeName: edpm-compute-0
    computeProfileName: dell-r740-shared-cpus
....
```
This CR is automatically created by nova-operator and the new
`ComputeProfileName` field is filled based on the
`nova.openstack.org/compute-profile` of the corresponding DataPlaneNode.

The `ComputeProfileName` field only holds a single profile name. So this
proposal does not allow composing ComputeProfiles. Composable ComputeProfiles
would require detailed definition of the precedence between the profiles and
we want to avoid that complexity here.

## Alternatives


## Implementation considerations

The dataplane-operator does not need to understand and handle the profile
annotations. That will be set by the human deployer and read by the service
operator the annotation refers to.

The new ComputeProfileName field will point to the ComputeProfile CRD for the
generic compute config. Changes in the DatalPlaneNode or Role compute-profile
annotation will be propagated to here. So assigning a different compute profile
to a Node is possible and it will mean a reconfiguration of the compute
service(s) on the EDPM node.

There is no need for controller reconciling ComputeProfile CRs as those CRs are
purely just data stores.