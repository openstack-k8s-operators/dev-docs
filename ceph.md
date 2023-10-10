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
     mgr 'allow rw' \
	 mon 'allow r' \
	 osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rwx pool=volumes, allow rwx pool=images'
```
If Manila is enabled in the OpenStack controlplane, then add `allow
rwx pool=cephfs.cephfs.meta, allow rwx pool=cephfs.cephfs.data` to the
end of the command above.

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

### Regarding multiple Ceph keyrings

There should only be one Ceph keyring file per Ceph cluster in the
`ceph-conf-files` secret. If OpenStack is being configured to use
multiple Ceph clusters, then the same `ceph-conf-files` secret
should have all of the Ceph backend files as in this example.
```yaml
apiVersion: v1
data:
  ceph.client.openstack.keyring: $KEY
  ceph.conf: $CONF
  ceph2.client.openstack.keyring: $KEY2
  ceph2.conf: $CONF2
kind: Secret
metadata:
  name: ceph-conf-files
  namespace: openstack
type: Opaque
```
The example above can be used to pass two configuration files and two
keyring files for the clusters known as "ceph" and "ceph2". When the
command `virsh secret-get-value $FSID` is passed the unique FSID
in `ceph.conf`, it will retrun the cephx secret key from
`ceph.client.openstack.keyring`. If the same command is passed the
unique FSID in `ceph2.conf`, it will retrun the cephx secret key from
`ceph2.client.openstack.keyring`. The FSID must map one-to-one with
the cephx secret. Thus, it is not supported to add an extra cephx
key called `ceph2.client.other.keyring` to the example above.

## Access the Ceph Secret via extraMounts

Use [extraMounts](extra_mounts.md) in CRs for pods which need to
access the Ceph secret. For example, the sample
[core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
has the following in the `OpenStackControlPlane` CR so that the
Glance and Cinder volume pods mount the cephx key and Ceph
configuration files in `/etc/ceph`.

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
The `OpenStackDataPlaneNodeSet` can also use `extraMounts`.
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
When a CR containing the above is created, an Ansible pod
running on OpenShift mounts the files in the Ceph secret
and copies them to the EDPM host using the
[edpm_ceph_client_files](https://github.com/openstack-k8s-operators/edpm-ansible/tree/main/roles/edpm_ceph_client_files)
Ansible role. The Nova containers then have a copy of `/etc/ceph`
that they can use to acccess the cephx key and Ceph configuration
file.

Though containers hosted on EDPM nodes use the path `/etc/ceph`
to access the Ceph client files, they are stored in
`/var/lib/openstack/config/ceph` on the EDPM nodes.

## Configure Glance

When Glance is configured with Ceph it's recommended to enable
[image conversion](https://github.com/openstack-k8s-operators/glance-operator/tree/main/config/samples/import_plugins).

Create a PVC to host a staging area using the example
[image_conversion_pvc.yaml](https://github.com/openstack-k8s-operators/glance-operator/blob/main/config/samples/import_plugins/image_conversion/image_conversion_pvc.yaml)
from the glance-operator repository.
```
oc create -f image_conversion_pvc.yaml
```
Change the storage size of the requested PVC to be as large as the largest
expected image after it has been converted into RAW format with a command
like `qemu-img convert -f qcow2 -O raw cirros.img cirros.raw`. When an image
is uploaded, Glance will use this space to convert it to RAW so that Ceph
can create volumes and VMs from the image efficiently using COW references.

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
        enabled_import_methods=[web-download,glance-direct]
        [glance_store]
        default_backend = default_backend
        [default_backend]
        rbd_store_ceph_conf = /etc/ceph/ceph.conf
        store_description = "RBD backend"
        rbd_store_pool = images
        rbd_store_user = openstack
        [image_import_opts]
        image_import_plugins = ['image_conversion']
        [image_conversion]
        output_format = raw
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
  playbook: osp.edpm.nova
```
The custom service is named `nova-custom-ceph`. It cannot be named
`nova` because `nova` is an immutable default service and will
overwrite any custom service with the same name during reconciliation.

After the `ConfigMap` and `OpenStackDataPlaneService` services above
have been created (e.g. `oc create -f ceph-nova.yaml`), update the
`OpenStackDataPlaneNodeSet`
[EDPM services list](https://openstack-k8s-operators.github.io/dataplane-operator/composable_services)
to replace the `nova` service with `nova-custom-ceph` and add the
`ceph-client` service.

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
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

## Configure Swift with a RGW backend

It's possible to configure an external Ceph Object Gateway (RGW) as an Object
Store service. Users can rely on the `Swift` client tool to interact with the
object store service.
To authenticate with the external RGW service, you must configure RGW to verify
users and their roles in the Identity service (keystone).
Run the following commands on an already deployed OpenStack control plane to
create users and roles as they will be used by the RGW instances to interact with
keystone.

```bash
openstack service create --name swift --description "OpenStack Object Storage" object-store
openstack user create --project service --password $SWIFT_PASSWORD swift
openstack role create swiftoperator
openstack role create ResellerAdmin
openstack role add --user swift --project service member
openstack role add --user swift --project service admin

export RGW_ENDPOINT=192.168.122.3
for i in public internal; do
    openstack endpoint create --region regionOne object-store $i http://$RGW_ENDPOINT:8080/swift/v1/AUTH_%\(tenant_id\)s;
done

openstack role add --project admin --user admin swiftoperator
```

- Replace `$SWIFT_PASSWORD` with the password that should be assigned to the
  `swift` user.
- Replace `192.168.122.3` with the IP address reserved as `$RGW_ENDPOINT`. If
  network isolation is used make sure the reserved address can be reached by
  the `swift` client that starts the connection.

The following commands should be run on a Ceph server to deploy and configure
a RGW service that will be able to serve object storage related requests.
Before deploying the RGW instances, the first step is to make sure Ceph has the
right `config-keys` required for the service to be able to interact with keystone.
Add the following configuration to the Ceph cluster via `ceph config set` commands:

```bash
cephadm shell
ceph config set global rgw_keystone_url "$KEYSTONE_ENDPOINT"
ceph config set global rgw_keystone_verify_ssl false
ceph config set global rgw_keystone_api_version 3
ceph config set global rgw_keystone_accepted_roles "member, Member, admin"
ceph config set global rgw_keystone_accepted_admin_roles "ResellerAdmin, swiftoperator"
ceph config set global rgw_keystone_admin_domain default
ceph config set global rgw_keystone_admin_project service
ceph config set global rgw_keystone_admin_user swift
ceph config set global rgw_keystone_admin_password "$SWIFT_PASSWORD"
ceph config set global rgw_keystone_implicit_tenants true
ceph config set global rgw_s3_auth_use_keystone true
ceph config set global rgw_swift_versioning_enabled true
ceph config set global rgw_swift_enforce_content_length true
ceph config set global rgw_swift_account_in_url true
ceph config set global rgw_trust_forwarded_https true
ceph config set global rgw_max_attr_name_len 128
ceph config set global rgw_max_attrs_num_in_req 90
ceph config set global rgw_max_attr_size 1024
```

- Replace `$KEYSTONE_ENDPOINT` with the identity service internal endpoint (the
  EDPM nodes are able to resolve the internal endpoint and not the public one)
- Replace `$SWIFT_PASSWORD` with the password assigned to the swift user in the
  previous step.

Deploy the RGW service and the associated ingress daemon:

```bash
cat <<EOF > "/tmp/rgw_spec"
---
service_type: rgw
service_id: rgw
service_name: rgw.rgw
placement:
  hosts:
    - $HOST1
    - $HOST2
    ...
    - $HOSTN
networks:
- $STORAGE_NETWORK
spec:
  rgw_frontend_port: 8082
  rgw_realm: default
  rgw_zone: default
---
service_type: ingress
service_id: rgw.default
service_name: ingress.rgw.default
placement:
  count: 1
spec:
  backend_service: rgw.rgw
  frontend_port: 8080
  monitor_port: 8999
  virtual_ip: $STORAGE_NETWORK_VIP
  virtual_interface_networks:
  - $STORAGE_NETWORK
```

- Replace `$HOST1`, `$HOST2`, ..., `$HOSTN` with the number of Ceph nodes
  where the RGW instances should be deployed;

- Replace `$STORAGE_NETWORK` with the network range used to resolve the
  interfaces where the radosgw processes should be bound;

- Replace `$STORAGE_NETWORK_VIP` with the `VIP` used as haproxy frontend: this
  address represents the `$RGW_ENDPOINT` that has been configured in the swift
  endpoint.


## Full Examples

The examples above are focussed on showing how a
single `OpenStackControlPlane` and `OpenStackDataPlaneNodeSet`
CR can be modified to include Ceph configuration by adding
`extraMounts` and `customServiceConfig`. Links to complete
examples are below.

- `OpenStackControlPlane`: [core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
- `OpenStackDataPlaneNodeSet`: [dataplane_v1beta1_openstackdataplanenodeset_ceph.yaml](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplanenodeset_ceph.yaml)
