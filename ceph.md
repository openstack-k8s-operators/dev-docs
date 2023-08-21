# Ceph

## Overview

This document describes how to use CRs from the
openstack-k8s-operators project to configure OpenStack
so that Glance, Cinder, and Nova use block storage
and Manila uses file storage from a Ceph cluster.
It does not require the use of install_yamls.

## Prerequisites

- Access to a Ceph cluster. If you intend to host Ceph on EDPM nodes,
  then follow the [Hyperconverged Infrastructure Documentation](hci.md)
  first.
- Access to an OpenShift cluster where the OpenStack operator is
  already running
- The reader knows how to create CRs from the openstack-k8s-operators
  project to deploy OpenStack

## Ensure Ceph is accessible via the Storage Network

As per the [Networking document](networking.md)
Ceph should be accessed via the storage network.
The storage network is the same as Ceph's public_network. The Glance
and Cinder pods should be able to access the storage network via
MetalLB. The Nova compute containers should also be able to access the
storage network. Glance, Cinder, Nova, and Manila will use a ceph.conf
file containing the IPs of the Ceph monitors; these IPs should be
within the storage network's IP range.  It is not necessary for
OpenStack to access Ceph's cluster_network.

## Create Ceph Pools for OpenStack

The commands in this section should be run on a Ceph server.

Create pools for Nova (vms), Cinder (volumes) and Glance (images).
```shell
for P in vms volumes images; do 
  cephadm shell -- ceph osd pool create $P;
  cephadm shell -- ceph osd pool application enable $P rbd;
done
```

If Manila is enabled in the OpenStack controlplane, create the `cephfs` volume.
```shell
cephadm shell -- ceph fs volume create cephfs
```

Create a cephx key which OpenStack can use to access the pools.
```shell
cephadm shell -- \
   ceph auth add client.openstack \ 
     mgr 'allow *' \
	 mon 'allow r' \
	 osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rwx pool=volumes, allow rwx pool=images'
```

Export the cephx key and Ceph configuration file.
```shell
cephadm shell -- ceph auth get client.openstack > /etc/ceph/ceph.client.openstack.keyring
cephadm shell -- ceph config generate-minimal-conf > /etc/ceph/ceph.conf
```

## Create a Ceph Secret

Transfer the cephx key and Ceph configuration file from the previous
section to a host which can create resources in the openstack
namespace. Base64 encode these files and store them in two variables.
```shell
KEY=$(cat /etc/ceph/ceph.client.openstack.keyring | base64 -w 0)
CONF=$(cat /etc/ceph/ceph.conf | base64 -w 0)
```
Use the variables to create a `ceph-conf-files` secret.
```shell
cat <<EOF > ceph_secret.yaml
apiVersion: v1
data:
  ceph.client.openstack.keyring: $KEY
  ceph.conf: $CONF
kind: Secret
metadata:
  name: ceph-conf-files
  namespace: openstack
type: Opaque
EOF

oc create -f ceph_secret.yaml
```

The Ceph FSID can be extracted from the secret using the following
command.
```shell
FSID=$(oc get secret ceph-conf-files -o json | jq -r '.data."ceph.conf"' | base64 -d | grep fsid | sed -e 's/fsid = //')
```
The above will be useful for configuration snippets covered later in
the document.

## Access the Ceph Secret via extraMounts

Use [extraMounts](extra_mounts.md) in CRs for pods which need to
access the Ceph secret. For example, the sample
[core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
has the following in the `OpenStackControlPlane` CR so that the
Glance and Cinder volume pods mount the cephx key and Ceph
configuration files in /etc/ceph.

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
spec:
  extraMounts:
    - name: v1
      region: r1
      extraVol:
        - propagation:
          - CinderVolume
          - GlanceAPI
          - ManilaShare
          extraVolType: Ceph
          volumes:
          - name: ceph
            projected:
              sources:
              - secret:
                  name: ceph-conf-files
          mounts:
          - name: ceph
            mountPath: "/etc/ceph"
            readOnly: true
```
The `OpenStackDataPlane` can also use `extraMounts`.
```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlane
spec:
  roles:
    edpm-compute:
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
When a CR containing the above is created, an Ansible pod
running on OpenShift mounts the files in the Ceph secret
and copies them to the EDPM host using the
[edpm_ceph_client_files](https://github.com/openstack-k8s-operators/edpm-ansible/tree/main/roles/edpm_ceph_client_files)
Ansible role. The Nova containers then have a copy of /etc/ceph
that they can use to acccess the cephx key and Ceph configuration
file.

## Configure Glance

Use a `customServiceConfig` to pass overrides to Glance's
configuration file. For example, the sample
[core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
has the following in the `OpenStackControlPlane` CR:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
spec:
  extraMounts:
    ...
  glance:
    template:
      databaseInstance: openstack
      customServiceConfig: |
        [DEFAULT]
        enabled_backends = default_backend:rbd
        [glance_store]
        default_backend = default_backend
        [default_backend]
        rbd_store_ceph_conf = /etc/ceph/ceph.conf
        store_description = "RBD backend"
        rbd_store_pool = images
        rbd_store_user = openstack
```

## Configure Cinder

Use a `customServiceConfig` to pass overrides to Cinder's
configuration file. For example, the sample
[core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
has the following in the `OpenStackControlPlane` CR:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
spec:
  extraMounts:
    ...
  cinder:
    template:
      cinderVolumes:
        ceph:
          customServiceConfig: |
            [DEFAULT]
            enabled_backends=ceph
            [ceph]
            volume_backend_name=ceph
            volume_driver=cinder.volume.drivers.rbd.RBDDriver
            rbd_ceph_conf=/etc/ceph/ceph.conf
            rbd_user=openstack
            rbd_pool=volumes
            rbd_flatten_volume_from_snapshot=False
            rbd_secret_uuid=$FSID
```
The `$FSID` value above should contain the actual FSID as described
in the "Create a Ceph Secret" section. The FSID itself does not need
to be considered secret.

## Configure Nova

Create a ConfigMap with content to be added to /etc/nova/nova.conf.d/,
inside the nova_compute container, so that Nova uses Ceph RBD.
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ceph-nova
data:
  03-ceph-nova.conf: |
    [libvirt]
    images_type=rbd
    images_rbd_pool=vms
    images_rbd_ceph_conf=/etc/ceph/ceph.conf
    images_rbd_glance_store_name=default_backend
    images_rbd_glance_copy_poll_interval=15
    images_rbd_glance_copy_timeout=600
    rbd_user=openstack
    rbd_secret_uuid=$FSID
```
The filename, e.g. 03-ceph-nova.conf, must match "*nova*.conf" and
the files are evaluated by Nova alphabetically (e.g. 01-foo-nova.conf
is processed before 02-bar-nova.conf).

The `$FSID` value above should contain the actual FSID as described
in the "Create a Ceph Secret" section. The FSID itself does not need
to be considered secret.

Create a custom version of the
[nova service](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/config/services/dataplane_v1beta1_openstackdataplaneservice_nova.yaml)
which ships with the dataplane operator so that it uses the ConfigMap
by adding it to the `configMaps` list.
```yaml
---
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneService
metadata:
  name: nova-custom-ceph
spec:
  label: dataplane-deployment-nova-custom-ceph
  configMaps:
    - ceph-nova
  secrets:
    - nova-cell1-compute-config
  role:
    name: "Deploy EDPM Nova-custom-ceph"
    hosts: "all"
    strategy: "linear"
    tasks:
      - name: "nova"
        import_role:
          name: "osp.edpm.edpm_nova"
        tags:
          - "edpm_nova"
```
The custom service is named `nova-custom-ceph`. It cannot be named
`nova` because `nova` is an immutable default service and will
overwrite any custom service with the same name during reconciliation.

After the `ConfigMap` and `OpenStackDataPlaneService` services above
have been created (e.g. `oc create -f ceph-nova.yaml`), update the
`OpenStackDataPlane`
[EDPM services list](https://openstack-k8s-operators.github.io/dataplane-operator/composable_services)
to replace the `nova` service with `nova-custom-ceph` and add the
`ceph-client` service.

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlane
spec:
  ...
  roles:
    edpm-compute:
      ...
      services:
        - configure-network
        - validate-network
        - install-os
        - configure-os
        - run-os
        - ceph-client
        - ovn
        - libvirt
        - nova-custom-ceph
```
The `ceph-client` service should be added before the `libvirt` and
`nova` (or `nova-custom-ceph` in this case) services. It configures
EDPM nodes as clients of a Ceph server by distributing the files,
which Ceph clients use, which were made available as described in the
"Create a Ceph Secret" section of the document.

When the `nova-custom-ceph` service Ansible job runs, it will copy
overrides from the ConfigMaps onto the Nova hosts. It will also use
`virsh secret-*` commands so that libvirt can retrieve the cephx
secret by FSID. This can be confirmed, after the job is completed, by
running the following on an EDPM node.

```
podman exec libvirt_virtsecretd virsh secret-get-value $FSID
```

## Configure Manila with native CephFS

Use a customServiceConfig to pass overrides to Manila's configuration
file. For example, the sample
[core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
has the following in the OpenStackControlPlane CR:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
spec:
  extraMounts:
    ...
    manila:
        template:
           manilaShares:
             share1:
                customServiceConfig: |
                    [DEFAULT]
                    enabled_share_backends=cephfs
                    enabled_share_protocols=cephfs
                    [cephfs]
                    driver_handles_share_servers=False
                    share_backend_name=cephfs
                    share_driver=manila.share.drivers.cephfs.driver.CephFSDriver
                    cephfs_conf_path=/etc/ceph/ceph.conf
                    cephfs_auth_id=openstack
                    cephfs_cluster_name=ceph
                    cephfs_enable_snapshots=True
                    cephfs_ganesha_server_is_remote=False
                    cephfs_volume_mode=0755
                    cephfs_protocol_helper_type=CEPHFS
```

## Full Examples

The examples above are focussed on showing how a
single `OpenStackControlPlane` and `OpenStackDataPlane`
CR can be modified to include Ceph configuration by adding
`extraMounts` and `customServiceConfig`. Links to complete
examples are below.

- `OpenStackControlPlane`: [core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
- `OpenStackDataPlane`: [dataplane_v1beta1_openstackdataplane_ceph.yaml](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplane_ceph.yaml)
