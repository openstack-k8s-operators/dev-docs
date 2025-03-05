# NFV OVS DPDK

This document describes how to deploy EDPM nodes with OVS DPDK.

## Procedure Overview

1. Configure kernel args and tuned parameters
2. Configure OVS DPDK parameters
3. Configure the networks with DPDK interfaces of the networker nodes
2. Configure Neutron parameters.

In order to complete the above procedure, the `services` list of the
`OpenStackDataPlaneNodeSet` CR needs to be edited.

## OpenStackDataPlaneNodeSet services list

Networker nodes can be configured by creating an
`OpenStackDataPlaneNodeSet` CR which the
[openstack-operator](https://openstack-k8s-operators.github.io/openstack-operator)
will reconcile to create OpenStackDataPlaneService resources
when an `OpenStackDataPlaneDeployment` CR is created.
OpenStackDataPlaneNodeSet CR has a `services` list like the following:

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  services:
    - bootstrap
    - download-cache
    - reboot-os
    - configure-ovs-dpdk
    - configure-network
    - validate-network
    - install-os
    - configure-os
    - ssh-known-hosts
    - run-os
    - install-certs
    - ovn
    - neutron-ovn
    - neutron-metadata
```
Only the services which are on the list will be configured.

## Configure the EDPM ansible variables of the Networker nodes

EDPM ansible variabes based on following EDPM ansible roles,
```
[edpm_ovs_dpdk](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_ovs_dpdk/meta/argument_specs.yml)
[edpm_kernel](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_kernel/meta/argument_specs.yml)
[edpm_tuned](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_tuned/meta/argument_specs.yml)
[edpm_ovn](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_ovn/meta/argument_specs.yml)
[edpm_libvirt](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_libvirt/meta/argument_specs.yml)

Following are the list of EDPM ansible variables which need to be
provided for deploying with OVS DPDK support.
```
edpm_ovs_dpdk_pmd_core_list: List of Logical CPUs to be allocated for Poll Mode Driver

edpm_ovs_dpdk_socket_memory: Socket memory list per NUMA node

edpm_ovs_dpdk_memory_channels: Number of memory channel per socket

edpm_ovn_bridge_mappings: List of bridge and dpdk ports mappings

edpm_kernel_args: Configures kernel boot parameters

edpm_tuned_profile: tuned profile configuration

edpm_tuned_isolated_cores: List of isolated cores
```

Example:
```
dpm_ovs_dpdk_pmd_core_list: "1,13,2,14,3,15"
edpm_ovs_dpdk_socket_memory: "4096"
edpm_ovs_dpdk_memory_channels: "4"
edpm_ovn_bridge_mappings: ['dpdk2:br-link2','dpdk1:br-link1']
edpm_kernel_args: "default_hugepagesz=1GB hugepagesz=1G hugepages=64 iommu=pt intel_iommu=on tsx=off isolcpus=2-11,14-23"
edpm_tuned_profile: "cpu-partitioning"
edpm_tuned_isolated_cores: "2-11,14-23"
```

edpm_network_config_template:
DPDK supported network interfaces should be specified in the network config templates to configure OVS DPDK on the Networker node.

```
edpm_network_config_template
  -
    type: ovs_user_bridge
    name: br-link
    use_dhcp: false
    members:
      -
        type: ovs_dpdk_port
        name: dpdk0
        mtu: 2000
        rx_queue: 2
        members:
          -
            type: interface
            name: nic3
```

This example also assumes that the Networker nodes:

- PXE and boot settings are already configured.
- Are at least one baremetal networker with DPDK interfaces.

Create an `OpenStackDataPlaneNodeSet` CR file,
e.g. `dataplane_cr.yaml` to represent the EDPM nodes. See
[dataplane_v1beta1_openstackdataplanenodeset.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplanenodeset.yaml)
for an example to modify as described in this document.

Do not yet create the CR in OpenShift as the edits described in the
next sections are required.

The example
[dataplane_v1beta1_openstackdataplanenodeset_ovs_dpdk.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/ovs_dpdk)
has OVS DPDK interfaces network configuration and required edpm ansible parameters.

Modify your `OpenStackDataPlaneNodeSet` CR to set
[edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible)
variables so that the
[edpm_network_config role](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_network_config/)
will configure networks with ovs dpdk interfaces.


`OpenStackDataPlaneNodeSet`
[Networker services list](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/#_composable_services)

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  roles:
    edpm-networker:
      ...
      services:
        - bootstrap
        - download-cache
        - reboot-os
        - configure-ovs-dpdk
        - configure-network
        - validate-network
        - install-os
        - configure-os
        - ssh-known-hosts
        - run-os
        - install-certs
        - ovn
        - neutron-ovn
        - neutron-metadata
```

### Create the OpenStackDataPlaneNodeSet

Create the CR from your directory based on the example
[dataplane_v1beta1_openstackdataplanenodeset_ovs_dpdk](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/ovs_dpdk)
with the changes described in the previous section.
```
oc kustomize --load-restrictor LoadRestrictionsNone openstack-operator/config/samples/dataplane/ovs_dpdk > dataplane_cr.yaml
```

### Create a OpenStackDataPlaneDeployment

Creating an `OpenStackDataPlaneDeployment` will trigger Ansible jobs
to configure an Networker. Which Ansible roles are run depends on the
`services` list.

Each `OpenStackDataPlaneDeployment` can have its own
`servicesOverride` list which will redefine the list
of services of an `OpenStackDataPlaneNodeSet` for a
deployment.

The example
[dataplane_v1beta1_openstackdataplanedeployment_ovs_dpdk](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/ovs_dpdk)

Create the CR based on the example
```
oc kustomize --load-restrictor LoadRestrictionsNone openstack-operator/config/samples/dataplane/ovs_dpdk > dataplane_cr.yaml
oc create -f dataplane_cr.yaml
```

The Netoworker OVS DPDK deployment should be complete after the Ansible jobs started
from creating the above CR finish successfully.
