# NFV OVS DPDK

This document describes how to deploy EDPM nodes with OVS DPDK.

## Procedure Overview

1. Configure kernel args and tuned parameters
2. Configure OVS DPDK parameters
3. Configure the networks with DPDK interfaces of the EDPM nodes
2. Configure Nova and Neutron parameters.

In order to complete the above procedure, the `services` list of the
`OpenStackDataPlaneNodeSet` CR needs to be edited.

## OpenStackDataPlaneNodeSet services list

EDPM nodes can be configured by creating an
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
    - libvirt
    - nova-custom-ovs-dpdk
```
Only the services which are on the list will be configured.

## Configure the EDPM ansible variables of the EDPM nodes

EDPM ansible variabes based on following EDPM ansible roles,
```
[edpm_ovs_dpdk](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_ovs_dpdk/meta/argument_specs.yml)
[edpm_kernel](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_kernel/meta/argument_specs.yml)
[edpm_tuned](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_tuned/meta/argument_specs.yml)
[edpm_ovn](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_ovn/meta/argument_specs.yml)
[edpm_libvirt](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_libvirt/meta/argument_specs.yml)
```
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

edpm_nova_libvirt_qemu_group: Configures qemu group for nova libvirt
```

Example:
```
dpm_ovs_dpdk_pmd_core_list: "1,13,2,14,3,15"
edpm_ovs_dpdk_socket_memory: "4096"
edpm_ovs_dpdk_memory_channels: "4"
edpm_ovn_bridge_mappings: ['dpdk2:br-link2','dpdk1:br-link1']
edpm_kernel_args: "default_hugepagesz=1GB hugepagesz=1G hugepages=64 iommu=pt intel_iommu=on tsx=off isolcpus=2-11,14-23"
edpm_nova_libvirt_qemu_group: "hugetlbfs"
edpm_tuned_profile: "cpu-partitioning"
edpm_tuned_isolated_cores: "2-11,14-23"
```

edpm_network_config_template:
DPDK supported network interfaces should be specified in the network config templates to configure OVS DPDK on the EDPM node.

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

This example also assumes that the EDPM nodes:

- PXE and boot settings are already configured.
- Are at least one baremetal compute with DPDK interfaces.

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


## Configure Nova

Create a ConfigMap with content to be added to /etc/nova/nova.conf.d/,
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ovs-dpdk-cpu-pinning-nova
data:
  04-cpu-pinning-nova.conf: |
    [DEFAULT]
    reserved_host_memory_mb = 4096
    [compute]
    cpu_shared_set = 0-3,24-27
    cpu_dedicated_set = 8-23,32-47
    [neutron]
    physnets = dpdk1, dpdk2
    [neutron_physnet_dpdk1]
    numa_nodes = 0
    [neutron_physnet_dpdk2]
    numa_nodes = 0
    [neutron_tunnel]
    numa_nodes = 0
```
The filename, e.g. `04-cpu-pinning-nova.conf`, must match `*nova*.conf` and
the files are evaluated by Nova alphabetically (e.g. `01-foo-nova.conf`
is processed before `02-bar-nova.conf`).

Create a custom version of the
[nova service](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/services/dataplane_v1beta1_openstackdataplaneservice_nova.yaml)
which ships with the dataplane operator so that it uses the ConfigMap
by adding it to the `configMaps` list.
```yaml
---
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneService
metadata:
  name: nova-custom-ovs-dpdk
spec:
  label: dataplane-deployment-nova-custom-ovs-dpdk
  configMaps:
    - ovs-dpdk-cpu-pinning-nova
  secrets:
    - nova-cell1-compute-config
  playbook: osp.edpm.nova
```
The custom service is named `nova-custom-ovs-dpdk`. It cannot be named
`nova` because `nova` is an immutable default service and will
overwrite any custom service with the same name during reconciliation.

After the `ConfigMap` and `OpenStackDataPlaneService` services above
have been created (e.g. `oc create -f nova-custom-ovs-dpdk.yaml`), update the
`OpenStackDataPlaneNodeSet`
[EDPM services list](https://openstack-k8s-operators.github.io/dataplane-operator/composable_services)
to replace the `nova` service with `nova-custom-ovs-dpdk`.

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  roles:
    edpm-compute:
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
        - libvirt
        - nova-custom-ovs-dpdk
```
The `bootstrap`, `configure-ovs-dpdk` and `configure-network` services
should be added before the `libvirt` and `nova` (or `nova-custom-ovs-dpdk`
in this case) services. It configures EDPM nodes with ovs dpdk.

When the `nova-custom-ovs-dpdk` service Ansible job runs, it will copy
overrides from the `ConfigMap`s onto the Nova hosts.

### Create the OpenStackDataPlaneNodeSet

Create the CR from your directory based on the example
[dataplane_v1beta1_openstackdataplanenodeset_ovs_dpdk](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/ovs_dpdk)
with the changes described in the previous section.
```
oc kustomize --load-restrictor LoadRestrictionsNone openstack-operator/config/samples/dataplaneovs_dpdk > dataplane_cr.yaml
```

### Create a OpenStackDataPlaneDeployment

Creating an `OpenStackDataPlaneDeployment` will trigger Ansible jobs
to configure an EDPM. Which Ansible roles are run depends on the
`services` list.

Each `OpenStackDataPlaneDeployment` can have its own
`servicesOverride` list which will redefine the list
of services of an `OpenStackDataPlaneNodeSet` for a
deployment.

The example
[dataplane_v1beta1_openstackdataplanedeployment_ovs_dpdk](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/ovs_dpdk)

Create the CR based on the example
```
oc kustomize --load-restrictor LoadRestrictionsNone openstack-operator/config/samples/dataplaneovs_dpdk > dataplane_cr.yaml
```

Custom `OpenStackDataPlaneService` called `nova-custom-ovs-dpdk`
has been created as described in
the [documentation to configure OpenStack to use OVS DPDK](ovs_dpdk.md).
The `nova-custom-ovs-dpdk` can be seen in the
[example](https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/examples/ovs_dpdk)
and takes the place of the default `nova`
OpenStackDataPlaneService. This custom service uses a `ConfigMap` called
`cpu-pinning-nova` which ensures that the file `03-cpu-pinning-nova.conf` is used
by Nova.

Now that the `nova-custom-ovs-dpdk` has been created, use the example
[dataplane_v1beta1_openstackdataplanedeployment_ovs_dpdk](https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/examples/ovs_dpdk)
to start the deployment.
```
oc kustomize --load-restrictor LoadRestrictionsNone openstack-operator/config/samples/dataplaneovs_dpdk > dataplane_cr.yaml
oc create -f dataplane_cr.yaml
```

The OVS DPDK deployment should be complete after the Ansible jobs started
from creating the above CR finish successfully.
