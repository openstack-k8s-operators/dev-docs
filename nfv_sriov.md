# NFV SRIOV

This document describes how to deploy EDPM nodes with SRIOV.

## Procedure Overview

1. Configure kernel args and tuned parameters
2. Configure SRIOV parameters
3. Configure the networks with SRIOV interfaces of the EDPM nodes
2. Configure Nova and Neutron parameters.

In order to complete the above procedure, the `services` list of the
`OpenStackDataPlaneNodeSet` CR needs to be edited.

## OpenStackDataPlaneNodeSet services list

EDPM nodes can be configured by creating an
`OpenStackDataPlaneNodeSet` CR which the
[dataplane-operator](https://openstack-k8s-operators.github.io/dataplane-operator)
will reconcile to create OpenStackDataPlaneService resources
when an `OpenStackDataPlaneDeployment` CR is created.
These types of CRs have a `services` list like the following:

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  services:
    - bootstrap
    - download-cache
    - configure-network
    - validate-network
    - install-os
    - configure-os
    - ssh-known-hosts
    - run-os
    - reboot-os
    - install-certs
    - libvirt
    - ovn
    - neutron-ovn
    - nova-custom-sriov
    - neutron-sriov
    - neutron-metadata
```
Only the services which are on the list will be configured.

## Configure the EDPM ansible variables of the EDPM nodes

Following are the list of EDPM ansible variables which need to be
provided for deploying with SRIOV support.
```
edpm_kernel_args: Configures kernel boot parameters

edpm_tuned_profile: tuned profile configuration

edpm_tuned_isolated_cores: List of isolated cores

edpm_nova_libvirt_qemu_group: Configures qemu group for nova libvirt
```

Example:
```
edpm_kernel_args: "default_hugepagesz=1GB hugepagesz=1G hugepages=64 iommu=pt intel_iommu=on tsx=off isolcpus=2-11,14-23"
edpm_tuned_profile: "cpu-partitioning"
edpm_tuned_isolated_cores: "2-11,14-23"
edpm_nova_libvirt_qemu_group: "hugetlbfs"
```

edpm_network_config_template:
SRIOV enabled network interfaces should be specified in the network config templates to configure SRIOV on the EDPM node.

```
edpm_network_config_template
  -
    type: sriov_pf
    name: nic3
    numvfs: 10
    use_dhcp: false
    promisc: true
```

This example also assumes that the EDPM nodes:

- PXE and boot settings are already configured.
- Are at least one baremetal compute with SRIOV interfaces.

Create an `OpenStackDataPlaneNodeSet` CR file,
e.g. `dataplane_cr.yaml` to represent the EDPM nodes. See
[dataplane_v1beta1_openstackdataplanenodeset.yaml](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplanenodeset.yaml)
for an example to modify as described in this document.

Do not yet create the CR in OpenShift as the edits described in the
next sections are required.

The example
[dataplane_v1beta1_openstackdataplanenodeset_sriov.yaml]https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/examples/sriov)
has SRIOV interfaces network configuration and required edpm ansible parameters.

Modify your `OpenStackDataPlaneNodeSet` CR to set
[edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible)
variables so that the
[edpm_network_config role](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_network_config/)
will configure networks with SRIOV interfaces.


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
```
The filename, e.g. `04-cpu-pinning-nova.conf`, must match `*nova*.conf` and
the files are evaluated by Nova alphabetically (e.g. `01-foo-nova.conf`
is processed before `02-bar-nova.conf`).

Create a custom version of the
[nova service](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/config/services/dataplane_v1beta1_openstackdataplaneservice_nova.yaml)
which ships with the dataplane operator so that it uses the ConfigMap
by adding it to the `configMaps` list.
```yaml
---
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneService
metadata:
  name: nova-custom-sriov
spec:
  label: dataplane-deployment-nova-custom-sriov
  configMaps:
    - ovs-dpdk-cpu-pinning-nova
  secrets:
    - nova-cell1-compute-config
  playbook: osp.edpm.nova
```
The custom service is named `nova-custom-ovsdpdk`. It cannot be named
`nova` because `nova` is an immutable default service and will
overwrite any custom service with the same name during reconciliation.

After the `ConfigMap` and `OpenStackDataPlaneService` services above
have been created (e.g. `oc create -f nova-custom-ovsdpdk.yaml`), update the
`OpenStackDataPlaneNodeSet`
[EDPM services list](https://openstack-k8s-operators.github.io/dataplane-operator/composable_services)
to replace the `nova` service with `nova-custom-ovsdpdk`.

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
        - configure-network
        - validate-network
        - install-os
        - configure-os
        - ssh-known-hosts
        - run-os
        - reboot-os
        - install-certs
        - libvirt
        - ovn
        - neutron-ovn
        - nova-custom-sriov
        - neutron-sriov
        - neutron-metadata
```
The `bootstrap`, `configure-network` and `neutron-sriov` services
should be added before the `libvirt` and `nova` (or `nova-custom-sriov`
in this case) services. It configures EDPM nodes with SRIOV.

When the `nova-custom-sriov` service Ansible job runs, it will copy
overrides from the `ConfigMap`s onto the Nova hosts.

### Create the OpenStackDataPlaneNodeSet

Create the CR from your directory based on the example
[dataplane_v1beta1_openstackdataplanenodeset_sriov](https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/examples/sriov)
with the changes described in the previous section.
```
oc kustomize --load-restrictor LoadRestrictionsNone dataplane-operator/examples/sriov > dataplane_cr.yaml
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
[dataplane_v1beta1_openstackdataplanedeployment_sriov](https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/examples/sriov)

Create the CR based on the example
```
oc kustomize --load-restrictor LoadRestrictionsNone dataplane-operator/examples/sriov > dataplane_cr.yaml
```

Custom `OpenStackDataPlaneService` called `nova-custom-sriov`
has been created as described in
the [documentation to configure OpenStack to use SRIOV](sriov.md).
The `nova-custom-sriov` can be seen in the
[example](https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/examples/sriov)
and takes the place of the default `nova`
OpenStackDataPlaneService. This custom service uses a `ConfigMap` called
`cpu-pinning-nova` which ensures that the file `03-cpu-pinning-nova.conf` is used
by Nova.

Now that the `nova-custom-sriov` has been created, use the example
[dataplane_v1beta1_openstackdataplanedeployment_sriov](https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/examples/sriov)
to start the second deployment.
```
oc kustomize --load-restrictor LoadRestrictionsNone dataplane-operator/examples/sriov > dataplane_cr.yaml
oc create -f dataplane_cr.yaml
```

The SRIOV deployment should be complete after the Ansible jobs started
from creating the above CR finish successfully.
