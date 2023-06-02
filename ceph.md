# Ceph

## Overview

This document describes how to use CRs from the
openstack-k8s-operators project to configure OpenStack
so that Glance, Cinder, and Nova use block storage
from an external Ceph cluster. It does not require
the use of install_yamls.

## Prerequisites

- Access to a Ceph cluster
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
storage network. Glance, Cinder, and Nova will use a ceph.conf file
containing the IPs of the Ceph monitors; these IPs should be within
the storage network's IP range.  It is not necessary for OpenStack to
access Ceph's cluster_network.

## Create Ceph Pools for OpenStack

The commands in this section should be run on a Ceph server.

Create pools for Nova (vms), Cinder (volumes) and Glance (images).
```
for P in vms volumes images; do 
  cephadm shell -- ceph osd pool create $P;
  cephadm shell -- ceph osd pool application enable $P rbd;
done
```

Create a cephx key which OpenStack can use to access the pools.
```
cephadm shell -- \
   ceph auth add client.openstack \ 
     mgr 'allow *' \
	 mon 'allow r' \
	 osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rwx pool=volumes, allow rwx pool=images'
```

Export the cephx key and Ceph configuration file.
```
cephadm shell -- ceph auth get client.openstack > /etc/ceph/ceph.client.openstack.keyring
cephadm shell -- ceph config generate-minimal-conf > /etc/ceph/ceph.conf
```

## Create a Ceph Secret

Transfer the cephx key and Ceph configuration file from the previous
section to a host which can create resources in the openstack
namespace. Base64 encode these files and store them in two variables.
```
KEY=$(cat /etc/ceph/ceph.client.openstack.keyring | base64 -w 0)
CONF=$(cat /etc/ceph/ceph.conf | base64 -w 0)
```
Use the variables to create a `ceph-conf-files` secret.
```
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
```
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

```
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
```
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

```
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

```
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

Use the `OpenStackDataPlane`
[NovaTemplate](https://openstack-k8s-operators.github.io/dataplane-operator/openstack_dataplanerole/#novatemplate)
to pass a `customServiceConfig`.

```
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlane
spec:
<snip>
  roles:
    edpm-compute:
    <snip>
        nova:
          cellName: cell1
          customServiceConfig: |
            [libvirt]
            images_type=rbd
            images_rbd_pool=vms
            images_rbd_ceph_conf=/etc/ceph/ceph.conf
            images_rbd_glance_store_name=default_backend
            images_rbd_glance_copy_poll_interval=15
            images_rbd_glance_copy_timeout=600
            rbd_user=openstack
            rbd_secret_uuid=$FSID
          deploy: true
          novaInstance: nova
```
The `$FSID` value above should contain the actual FSID as described
in the "Create a Ceph Secret" section. The FSID itself does not need
to be considered secret.

## Full Examples

The examples above are focussed on showing how a
single `OpenStackControlPlane` and `OpenStackDataPlane`
CR can be modified to include Ceph configuration by adding
`extraMounts` and `customServiceConfig`. Links to complete
examples are below.

- `OpenStackControlPlane`: [core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
- `OpenStackDataPlane`: [dataplane_v1beta1_openstackdataplane_ceph.yaml](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplane_ceph.yaml)
