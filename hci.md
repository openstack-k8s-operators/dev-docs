# Hyperconverged Infrastructure

This document describes how to deploy EDPM nodes which
host both Ceph Storage and Nova Compute services. These types of
deployments are also known as Hyperconverged Infrastructure (HCI).

## Procedure Overview

1. Configure the networks of the EDPM nodes
2. Install Ceph on EDPM nodes
3. Configure OpenStack to use the collocated Ceph server

In order to complete the above procedure, the `services` list of the
`OpenStackDataPlaneNodeSet` CR needs to be edited.

## OpenStackDataPlaneNodeSet services list

EDPM nodes can be configured by creating an
`OpenStackDataPlaneNodeSet` CR which the
[dataplane component of the openstack operator](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/)
will reconcile when an `OpenStackDataPlaneDeployment` CR is created.
These types of CRs have a `services` list like the following:

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  services:
    - bootstrap
    - configure-network
    - validate-network
    - install-os
    - ceph-hci-pre
    - configure-os
    - ssh-known-hosts
    - run-os
    - reboot-os
```
Only the services which are on the list will be configured.

Because we need to deploy Ceph on an EDPM node after the storage
network and NTP are configured but before Nova is configured, we
need to have two `OpenStackDataPlaneDeployments` each with their own
`services` list.

## Configure the networks of the EDPM nodes

This example assumes that the Control Plane has been deployed but
has not yet been modified to use Ceph (because the Ceph cluster does
not yet exist).

This example also assumes that the EDPM nodes:

- Have already been provisioned with an operating system (RHEL or CentOS)
- Are accessible via an SSH key that Ansible can use
- Have disks available to be used as Ceph OSDs
- Are at least three in number (Ceph clusters must have at least three
  nodes for redundancy)

Create an `OpenStackDataPlaneNodeSet` CR file,
e.g. `dataplane_cr.yaml` to represent the EDPM nodes. See
[dataplane_v1beta1_openstackdataplanenodeset.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplanenodeset.yaml)
for an example to modify as described in this document.

Do not yet create the CR in OpenShift as the edits described in the
next sections are required.

### Add a Ceph cluster network

Ceph normally uses two networks:

- `storage` - Storage traffic, the Ceph `public_network`, e.g. Glance,
  Cinder and Nova containers use this network for RBD traffic to the
  Ceph cluster. Block (RBD) storage clients of Ceph need access to
  this network.

- `storage_mgmt` - Storage management traffic (such as replication
  traffic between storage nodes), the Ceph `cluster_network`,
  e.g. Ceph OSDs use this network to replicate data. This network
  is used by Ceph OSD servers but not Ceph clients.

The [Networking Documentation](networking.md) covers the `storage`
network since pods in OpenShift and containers on RHEL needs to access
the storage network. It does not cover the `storage_mgmt` network
since that network is used exclusively by Ceph.

The example
[dataplane_v1beta1_openstackdataplanenodeset_pre_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/pre_ceph_hci) and
[dataplane_v1beta1_openstackdataplanenodeset_post_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/post_ceph_hci)
has both the `storage` and `storage_mgmt` networks since those EDPM
nodes will host Ceph OSDs.

Modify your `OpenStackDataPlaneNodeSet` CR to set
[edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible)
variables so that the
[edpm_network_config role](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/roles/edpm_network_config/)
will configure a storage management network which Ceph will use as a
cluster network. For this example we'll assume that the storage
management network range is `172.20.0.0/24` and that it is on `VLAN23`.

### MTU Settings for Ceph

The example
[dataplane_v1beta1_openstackdataplanenodeset_pre_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/pre_ceph_hci) and
[dataplane_v1beta1_openstackdataplanenodeset_post_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/post_ceph_hci)
changes the MTU of the `storage` and `storage_mgmt`
network from `1500` to `9000` (jumbo frames) for improved storage
performance (though it is not mandatory to increase the MTU). If jumbo
frames are used, then all network switch ports in the data path must
be configured to support jumbo frames and MTU changes must also be
made for pods using the storage network running on OpenShift.

To change the MTU for the OpenShift pods connecting to the dataplane
nodes, update the Node Network Configuration Policy (NNCP) for the
base interface as well as the VLAN interface. It is not necessary
to update the Network Attachment Definition (NAD) if the main NAD
interface already has the desired MTU. If the MTU of the underlying
interface is set to 9000 and it isn't specified for the VLAN interface
on top of it, then it will default to the value from the underlying
interface. See the
[CNI macvlan plugin documentation](https://www.cni.dev/plugins/current/main/macvlan).
For information on the NNCP and NAP see the
[networking documentation](networking.md).

If the MTU values are not consistent then problems may manifest on the
application layer that could cause the Ceph cluster to not reach
quorum or not support authentication using the CephX protocol. If the
MTU is changed, and these types of problems are observed, then
verify that all hosts using the network using jumbo frames can
communicate at the desired MTU with a command like `ping -M do -s 8972
172.20.0.100`.

### Create the OpenStackDataPlaneNodeSet

Create the CR from your file based on the example
[dataplane_v1beta1_openstackdataplanenodeset_pre_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/pre_ceph_hci) and
[dataplane_v1beta1_openstackdataplanenodeset_post_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/post_ceph_hci)
with the changes described in the previous section.
```
oc kustomize --load-restrictor LoadRestrictionsNone config/samples/dataplane/pre_ceph_hci | oc apply -f -
oc kustomize --load-restrictor LoadRestrictionsNone config/samples/dataplane/post_ceph_hci | oc apply -f -

```

### Create a pre-Ceph OpenStackDataPlaneDeployment

Creating an `OpenStackDataPlaneDeployment` will trigger Ansible jobs
to configure an EDPM. Which Ansible roles are run depends on the
`services` list.

Each `OpenStackDataPlaneDeployment` can have its own
`servicesOverride` list which will redefine the list
of services of an `OpenStackDataPlaneNodeSet` for a
deployment.

The example
[dataplane_v1beta1_openstackdataplanedeployment_pre_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/pre_ceph_hci)
has a shortened list of services which need to be configured
before Ceph is deployed on an EDPM node in an HCI scenario.

Create the CR based on the example
```
oc create -f openstackdataplanedeployment_pre_ceph_hci.yaml
```

#### Pre-Ceph Service Details

The example
[dataplane_v1beta1_openstackdataplanedeployment_pre_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/pre_ceph_hci)
contains the `ceph-hci-pre` service. This service
prepares EDPM nodes to host Ceph services
after the network has been configured. It does this by running the
edpm-ansible role called `ceph-hci-pre`. This role injects a
`ceph-networks.yaml` file into `/var/lib/edpm-config/firewall`
so that when the `edpm_nftables` role runs, firewall ports are open
for Ceph services. By default the `ceph-networks.yaml` file only
contains directives to open the ports required by the Ceph RBD
(block), RGW (object) and NFS (files) services. This is because of the
following default Ansible variable value:
```yaml
edpm_ceph_hci_pre_enabled_services:
  - ceph_mon
  - ceph_mgr
  - ceph_osd
  - ceph_rgw
  - ceph_nfs
  - ceph_rgw_frontend
  - ceph_nfs_frontend
```
If other Ceph services, like the Ceph Dashboard, will be deployed
on HCI nodes, then add additional services to the enabled services
list above. For more information, see the `ceph-hci-pre` role in the
[edpm-ansible role documentation](https://openstack-k8s-operators.github.io/edpm-ansible/roles.html).

As seen in the example
[dataplane_v1beta1_openstackdataplanedeployment_pre_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples/dataplane/pre_ceph_hci),
the `configure-os` and `run-os` services are run after `ceph-hci-pre`
because they enable the firewall rules which `ceph-hci-pre` put in
place. The `run-os` service also configures NTP, which is required by
Ceph.

### Confirm the Network is configured

Before proceeding to the next section confirm this section has been
completed by doing the following.

- SSH into an EDPM node
- Use the `ip a` command to display the configured networks
- Confirm that the storage networks are in the list of configured networks

## Install Ceph on EDPM nodes

Use [cephadm](https://docs.ceph.com/en/latest/cephadm/index.html)
to install Ceph on the EDPM nodes. The `cephadm` package will need
to be installed on at least one EDPM node first.
[edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible)
does not install Ceph. Have `cephadm` use the storage
network and tune Ceph for HCI as described in the next sections.
Resume use of the OpenStack tools described
in this document after Ceph has been installed on the EDPM nodes with
Ceph tools. The rest of this section describes Ceph configuration to
be applied will using `cephadm` including use of the storage networks
configured in the previous section and Ceph tuning for HCI.

### Configuring Ceph to use the Storage Networks

Use the storage network as described in [Networking](networking.md)
for the Ceph `public_network`. Use the storage management network
for the Ceph `cluster_network`. For example, if the storage network
is `172.18.0.0/24` and the storage management network is
`172.20.0.0/24`, then create an initial `ceph.conf` file with the
following:
```ini
[global]
public_network = 172.18.0.0/24
cluster_network = 172.20.0.0/24
```
The initial `ceph.conf` file can be passed when `cephadm bootstrap` is
run with the `--config` option.

Pass the Storage IP of the EDPM node being bootstrapped with the
`--mon-ip` option. For example, if the EDPM node where Ceph will be
bootstrapped has the Storage IP `172.18.0.100`, then it is passed like
this.
```shell
cephadm bootstrap --config ceph.conf --mon-ip 172.18.0.100 ...
```

### Tuning Ceph for collocation with Nova Services

When collocating Nova Compute and Ceph OSD services, boundaries can be
set to reduce contention for CPU and Memory between the two
services. To limit Ceph for HCI, use an the initial `ceph.conf`
file with the following.

```ini
[osd]
osd_memory_target_autotune = true
osd_numa_auto_affinity = true
[mgr]
mgr/cephadm/autotune_memory_target_ratio = 0.2
```
The
[osd_memory_target_autotune](https://docs.ceph.com/en/latest/cephadm/services/osd/#automatically-tuning-osd-memory)
is set to true so that the OSD daemons will adjust their memory
consumption based on the `osd_memory_target` config option. The
`autotune_memory_target_ratio` defaults to 0.7. So 70% of the total RAM
in the system is the starting point, from which any memory consumed by
non-autotuned Ceph daemons are subtracted, and then the remaining
memory is divided by the OSDs (assuming all OSDs have
`osd_memory_target_autotune` true). For HCI deployments the
`mgr/cephadm/autotune_memory_target_ratio` can be set to 0.2 so that
more memory is available for the Nova Compute service.

A two NUMA node system can host a latency sensitive Nova workload on
one NUMA node and a Ceph OSD workload on the other NUMA node. To
configure Ceph OSDs to use a specific NUMA node (and not the one being
used by the Nova Compute workload) use either of the following Ceph
OSD configurations.

- `osd_numa_node` sets affinity to a numa node (`-1` for none)
- `osd_numa_auto_affinity` automatically sets affinity to the NUMA
  node where storage and network match

If there are network interfaces on both NUMA nodes and the disk
controllers are on NUMA node `0`, then use a network interface on NUMA
node `0` for the storage network and host the Ceph OSD workload on NUMA
node `0`. Then host the Nova workload on NUMA node `1` and have it use
the network interfaces on NUMA node `1`. Setting
`osd_numa_auto_affinity`, to true, as in the initial `ceph.conf` file
above, should result in this configuration. Alternatively, the
`osd_numa_node` could be set directly to `0` and
`osd_numa_auto_affinity` could be unset so that it will default to
false.

When a hyperconverged cluster backfills as a result of an OSD going
offline, the backfill process can be slowed down. In exchange for a
slower recovery, the backfill activity has less of an impact on
the collocated Compute workload. Ceph has the following defaults to
control the rate of backfill activity.

```ini
osd_recovery_op_priority = 3
osd_max_backfills = 1
osd_recovery_max_active_hdd = 3
osd_recovery_max_active_ssd = 10
```

It is not necessary to pass the above in an initial `ceph.conf` as
they are the default values, but if these values need to be deployed
with different values modify an example like the above and add it to
the initial Ceph configuration file before deployment. If the values
need to be adjusted after the deployment use `ceph config set osd
<key> <value>`.

### Confirm Ceph is deployed

Before proceeding to the next section confirm this section has been
completed by doing the following.

- SSH into an EDPM node
- Use the `cephadm shell -- ceph -s` command to see status of the Ceph
  cluster

#### Confirm Ceph is tuned

Use `cephadm shell` to start a Ceph shell and confirm the tuning
values were applied. For example, to check that the NUMA and memory
target auto tuning run commands like this:
```shell
  [ceph: root@edpm-compute-0 /]# ceph config dump | grep numa
    osd                                             advanced  osd_numa_auto_affinity                 true
  [ceph: root@edpm-compute-0 /]# ceph config dump | grep autotune
    osd                                             advanced  osd_memory_target_autotune             true
  [ceph: root@edpm-compute-0 /]# ceph config get mgr mgr/cephadm/autotune_memory_target_ratio
  0.200000
  [ceph: root@edpm-compute-0 /]#
```

We can then confirm that a specific OSD, e.g. osd.11, inherited those
values with commands like this:

```shell
  [ceph: root@edpm-compute-0 /]# ceph config get osd.11 osd_memory_target
  4294967296
  [ceph: root@edpm-compute-0 /]# ceph config get osd.11 osd_memory_target_autotune
  true
  [ceph: root@edpm-compute-0 /]# ceph config get osd.11 osd_numa_auto_affinity
  true
  [ceph: root@edpm-compute-0 /]#
```

To confirm that the default backfill values are set for the same
example OSD, use commands like this:

```shell
  [ceph: root@edpm-compute-0 /]# ceph config get osd.11 osd_recovery_op_priority
  3
  [ceph: root@edpm-compute-0 /]# ceph config get osd.11 osd_max_backfills
  1
  [ceph: root@edpm-compute-0 /]# ceph config get osd.11 osd_recovery_max_active_hdd
  3
  [ceph: root@edpm-compute-0 /]# ceph config get osd.11 osd_recovery_max_active_ssd
  10
  [ceph: root@edpm-compute-0 /]#
```

## Configure OpenStack to use the collocated Ceph server

Follow the
[documentation to configure OpenStack to use Ceph](ceph.md).
Though the Ceph cluster is physically co-located on the EDPM
nodes, which will also host the compute services, it can be treated
as if it is logically external. The [documentation](ceph.md) will
cover how to configure the Control Plane and Data Plane to use
Ceph. Ensure that the services list is updated accordingly. When
configuring the Data Plane there are additional steps required when
using HCI which are covered below.

### Update the Data Plane CR

One way to update DataPlane CR(s) is to use the `oc edit` command:

```
oc edit openstackdataplane.dataplane.openstack.org
```

The sections below describe modifications to be made to the Data Plane
CR in detail. After the edits described below are completed, the
operators will reconcile the new configuration.

### Add ExtraMounts

The
[documentation to configure OpenStack to use Ceph](ceph.md)
includes using [extraMounts](extra_mounts.md). If you add Nova
overrides as described below without adding `extraMounts`, then Nova
service configuration will fail because of missing a missing CephX key
and Ceph configuration file.

<!-- thus extraMounts is mentioned first in case the user stops the oc edit early -->

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  nodeTemplate:
    extraMounts:
    - extraVolType: Ceph
      volumes:
      - name: ceph
        secret:
          secretName: ceph-conf-files
      mounts:
      - name: ceph
        mountPath: "/etc/ceph"
        readOnly: true
```

### Create a post-Ceph OpenStackDataPlaneDeployment

Create a second `OpenStackDataPlaneDeployment` which will trigger the
Ansible jobs to complete the EDPM Compute node configuration.

The example
[dataplane_v1beta1_openstackdataplanedeployment_post_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplanedeployment_post_ceph_hci.yaml)
has a shortened list of services which need to be configured
after Ceph is deployed on an EDPM node in an HCI scenario.

Before creating the deployment-post-ceph CR ensure that a
custom `OpenStackDataPlaneService` called `nova-custom-ceph`
has been created as described in
the [documentation to configure OpenStack to use Ceph](ceph.md).
The `nova-custom-ceph` can be seen in the
[example](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplanedeployment_post_ceph_hci.yaml)
and takes the place of the default `nova`
OpenStackDataPlaneService. This custom service uses a ConfigMap called
`ceph-nova` which ensures that the file `03-ceph-nova.conf` is used
by Nova.

Create an additional ConfigMap to set the `reserved_host_memory_mb`
to a value appropriate for your system.

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: reserved-memory-nova
data:
  04-reserved-memory-nova.conf: |
    [DEFAULT]
    reserved_host_memory_mb=75000
```

The value for the `reserved_host_memory_mb` may be set so that the
Nova scheduler does not give memory to a virtual machine that a Ceph
OSD on the same server will need. The example above reserves 5 GB per
OSD for 10 OSDs per host in addition to the default reserved memory
for the hypervisor. In an IOPS-optimized cluster performance can be
improved by reserving more memory per OSD.  The 5 GB number is
provided as a starting point which can be further tuned if necessary.

Use `oc edit OpenStackDataPlaneService/nova-custom-ceph` to add
`reserved-memory-nova` to the `configMaps` list.

```yaml
---
kind: OpenStackDataPlaneService
<...>
spec:
  configMaps:
  - ceph-nova
  - reserved-memory-nova
```

Now that the `nova-custom-ceph` has been created, use the example
[dataplane_v1beta1_openstackdataplanedeployment_post_ceph_hci.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplanedeployment_post_ceph_hci.yaml)
to start the second deployment.

```
oc create -f openstackdataplanedeployment_pre_post_hci.yaml
```

The HCI deployment should be complete after the Ansible jobs started
from creating the above CR finish successfully.

## Final OpenStackDataPlaneNodeSet services list

It is important to restore the full `services` list in the
`OpenStackDataPlaneNodeSet` so that during updates all required
services are updated.

Before Ceph was deployed the initial services list looked like this:

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  services:
    - bootstrap
    - configure-network
    - validate-network
    - install-os
    - ceph-hci-pre
    - configure-os
    - ssh-known-hosts
    - run-os
    - reboot-os
```

After Ceph was deployed the initial services list looked like this:

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  services:
    - install-certs
    - ceph-client
    - ovn
    - neutron-metadata
    - libvirt
    - nova-custom-ceph
```

Now we need to update the final services list of the HCI nodes to
combine both lists like this:

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
spec:
  ...
  services:
    - bootstrap
    - configure-network
    - validate-network
    - install-os
    - ceph-hci-pre
    - configure-os
    - ssh-known-hosts
    - run-os
    - reboot-os
    - install-certs
    - ceph-client
    - ovn
    - neutron-metadata
    - libvirt
    - nova-custom-ceph
```
Updating the services list in a `OpenStackDataPlaneNodeSet` will not
trigger another run of Ansible unless a new
`OpenStackDataPlaneDeployment` is created. However, we want to make
sure the `OpenStackDataPlaneNodeSet` has the complete list of services
for future deployments.
