## External Ceph - ROOK / CRC

This guide outlines how to set up an external Ceph cluster using the ROOK
operator in a CRC (CodeReady Containers) or SNO (Single Node OpenShift)
environment. The instructions assume you're using the
[install_yamls](https://github.com/openstack-k8s-operators/install_yamls)
project for development tools.

The setup involves 2 nodes:
- a `crc` environment that can be replaced by a `SNO`
- an `EDPM` node (hosting `Ceph`)

### Topology diagram

```
     +----------+        +-----------+
     |          |        |           |- 192.168.122.100 (provisioning)
     | crc/sno  | <====> | EDPM/Ceph |
     |          |        |           |- 172.18.0.100 (storage)
     +----------+        +-----------+
           |     multus      |     |_(optional) 192.168.130.12 (crc bridge)
           +-----------------+
```

**Note**:
The optional crc bridge (192.168.130.12) may be used in **not** isolated
setups.

### Prerequisites

Follow the instructions in the [install_yamls](https://github.com/openstack-k8s-operators/install_yamls)
Readme to start the OpenStack control plane and deploy ROOK.

```
make crc_storage
make attach_default_interface # SKIP THIS IF NETWORK_ISOLATION=false
make openstack
make openstack_deploy
make rook
```

Once the `OpenStack` control plane is up and the `ROOK` operator is deployed, a
`rook-ceph` namespace is created. Now prepare the `EDPM` node to host the
single-node `Ceph` cluster:

```bash
cd install_yamls/devsetup
make edpm_compute
```

#### Handling Network isolation

If `NETWORK_ISOLATION=false`, attach the `crc` bridge to the `EDPM` node:

```
sudo virsh attach-interface --domain edpm-compute-0 --type bridge --source crc --model virtio --config --live
```

Otherwise, skip this step.


### Install dependencies on the EDPM node

Ensure the EDPM node (running CentOS 9-stream) has the required dependencies:

```bash
ssh -o StrictHostKeyChecking=no -i ~/install_yamls/out/edpm/ansibleee-ssh-key-id_rsa cloud-admin@192.168.122.100
sudo dnf install -y lvm2 git curl podman epel-release
sudo dnf install -y centos-release-ceph-reef
sudo dnf install -y ceph-common
```

### Network and deploy the EDPM node

For non-isolated environments, add an `IP address` to the `crc bridge` and ping
the network:

```bash
sudo ip a add 192.168.130.12/24 dev eth1
ping -c3 192.168.130.11
```

In network-isolated environment, simply deploy the `EDPM` node using the
existing install_yamls target.

**Note:**
The rest of the guide assumes that you're working in a network isolation
context and `Ceph` should be deployed using the `EDPM` node storage network,
propagated to the ctlplane via nncp / net-attach-def.

```
make edpm_deploy
```

### Deploy Ceph

In a network-isolated context, verify the `EDPM` node is deployed and `AEE`
completed the execution:

```bash
NAME                                                          READY   STATUS      RESTARTS   AGE
bootstrap-edpm-deployment-openstack-edpm-ipam-hbhnh           0/1     Completed   0          9h
configure-network-edpm-deployment-openstack-edpm-ipam-6gzwq   0/1     Completed   0          9h
configure-os-edpm-deployment-openstack-edpm-ipam-bdzwx        0/1     Completed   0          9h
download-cache-edpm-deployment-openstack-edpm-ipam-bbqll      0/1     Completed   0          9h
install-certs-edpm-deployment-openstack-edpm-ipam-cnzgf       0/1     Completed   0          9h
install-os-edpm-deployment-openstack-edpm-ipam-bfgl4          0/1     Completed   0          9h
libvirt-edpm-deployment-openstack-edpm-ipam-98zlt             0/1     Completed   0          9h
neutron-metadata-edpm-deployment-openstack-edpm-ipam-xt8gg    0/1     Completed   0          9h
nova-edpm-deployment-openstack-edpm-ipam-kprc6                0/1     Completed   0          8h
ovn-edpm-deployment-openstack-edpm-ipam-mpv7q                 0/1     Completed   0          9h
reboot-os-edpm-deployment-openstack-edpm-ipam-mjhmw           0/1     Completed   0          9h
repo-setup-edpm-deployment-openstack-edpm-ipam-q5k55          0/1     Completed   0          9h
run-os-edpm-deployment-openstack-edpm-ipam-t4q9p              0/1     Completed   0          9h
ssh-known-hosts-edpm-deployment-j5gqk                         0/1     Completed   0          9h
telemetry-edpm-deployment-openstack-edpm-ipam-bhz7j           0/1     Completed   0          8h
validate-network-edpm-deployment-openstack-edpm-ipam-t7rh9    0/1     Completed   0          9h
```

1. Ensure the `vlan21` interface (e.g., 172.18.0.100) is on the storage
   network.

```
$ ip -o -4 a

1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
4: br-ex    inet 192.168.122.100/24 brd 192.168.122.255 scope global br-ex\       valid_lft forever preferred_lft forever
5: vlan20    inet 172.17.0.100/24 brd 172.17.0.255 scope global vlan20\       valid_lft forever preferred_lft forever
6: vlan21    inet 172.18.0.100/24 brd 172.18.0.255 scope global vlan21\       valid_lft forever preferred_lft forever
7: vlan22    inet 172.19.0.100/24 brd 172.19.0.255 scope global vlan22\       valid_lft forever preferred_lft forever
11: podman0    inet 10.255.255.1/24 brd 10.255.255.255 scope global podman0\       valid_lft forever preferred_lft forever
```

2. Build one or more OSD devices:

```bash
cd devsetup/ceph
./rebuild_osd.sh 3 # it builds an OSD on top of loop3
```

3. Verify the OSD device:

```bash
$ sudo lvs
LV           VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
ceph_lv_data ceph_vg_3 -wi-a----- <7.00g
```

4. Deploy Ceph:

```bash
./deploy.sh -i 172.18.0.100 -c quay.io/ceph/ceph:v18 -p rook:rbd -d /dev/ceph_vg_3/ceph_lv_data
```

5. Review the Ceph deployment logs and follow the "Next Steps":

```bash
...
...
...

[INSTALL EXTERNAL_ROOK SCRIPTS] - Rook scripts are ready
Extract Ceph cluster details
export ARGS="[Configurations]
ceph-conf = /etc/ceph/ceph.conf
keyring = /etc/ceph/ceph.client.admin.keyring
run-as-user = client.rook
rgw-pool-prefix = default
format = bash
output = /home/cloud-admin/rook-env-vars.sh
rbd-data-pool-name = rook
"
export ROOK_EXTERNAL_FSID=<redacted>
export ROOK_EXTERNAL_USERNAME=client.rook
export ROOK_EXTERNAL_CEPH_MON_DATA=edpm-compute-0=172.18.0.100:6789
export ROOK_EXTERNAL_USER_SECRET=<redacted>
export CSI_RBD_NODE_SECRET=<redacted>
export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node
export CSI_RBD_PROVISIONER_SECRET=<redacted>
export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner
export MONITORING_ENDPOINT=172.18.0.100
export MONITORING_ENDPOINT_PORT=9283
export RBD_POOL_NAME=rook
export RGW_POOL_PREFIX=default

...

External ROOK - Next steps:
1. Copy the rook-env-vars script from /home/cloud-admin/rook-env-vars.sh to the OpenShift client
2. Get [import-external-cluster.sh](https://raw.githubusercontent.com/rook/rook/refs/heads/master/deploy/examples/import-external-cluster.sh) script
3. On the OpenShift client node, run: source rook-env-vars.sh && ./import-external-cluster.sh
```

**Note:**

We're not interested in building multiple pools for openstack in this case, so
we're only creating the `rook` pool and the `client.rook` user. If you're
planning to connect Ceph to the OpenStack control plane, ndd more pools with
the following notation: `-p <pool>:rbd` (e.g., -p volumes:rbd -p images:rbd -p
vms:rbd).

### Import Ceph into Rook

1. Copy the rook-env-vars.sh to the `OCP` client node and download the
   `import-external-cluster` script:

```
scp -o StrictHostKeyChecking=no -i install_yamls/out/edpm/ansibleee-ssh-key-id_rsa \
    cloud-admin@192.168.122.100:/home/cloud-admin/rook-env-vars.sh .

```

2. Run the script and check the produced output:

```
source rook-env-vars.sh && ./import-external-cluster.sh
..
..
cluster namespace rook-ceph already exists
secret/rook-ceph-mon created
configmap/rook-ceph-mon-endpoints created
configmap/external-cluster-user-command created
secret/rook-csi-rbd-node created
secret/rook-csi-rbd-provisioner created
storageclass.storage.k8s.io/ceph-rbd created
```

### Patch ROOK Operator for Storage Network


1. Apply `NetworkAttachmentDefinition` CR to make the `Storage` and `StorageMgmt`
   networks available in the `rook-ceph` namespace:

```bash
apiVersion: v1
kind: List
items:
- apiVersion: k8s.cni.cncf.io/v1
  kind: NetworkAttachmentDefinition
  metadata:
    name: storage
    namespace: rook-ceph
  spec:
    config: |
      {
        "cniVersion": "0.3.1",
        "name": "storage",
        "type": "macvlan",
        "master": "enp6s0.21",
        "ipam": {
          "type": "whereabouts",
          "range": "172.18.0.0/24",
          "range_start": "172.18.0.30",
          "range_end": "172.18.0.70"
        }
      }
- apiVersion: k8s.cni.cncf.io/v1
  kind: NetworkAttachmentDefinition
  metadata:
    name: storagemgmt
    namespace: rook-ceph
  spec:
    config: |
      {
        "cniVersion": "0.3.1",
        "name": "storagemgmt",
        "type": "macvlan",
        "master": "enp6s0.23",
        "ipam": {
          "type": "whereabouts",
          "range": "172.20.0.0/24",
          "range_start": "172.20.0.30",
          "range_end": "172.20.0.70"
        }
      }
```

2. Patch the `ROOK` operator to access the `Storage` network:

```
oc --namespace rook-ceph patch deployment rook-ceph-operator --type=json -p='[{"op": "add", "path": "/spec/template/metadata/annotations/k8s.v1.cni.cncf.io~1networks", "value": "storage"}]'
```

3. Restart the `ROOK` operator if necessary:

```
oc rollout restart deployment rook-ceph-operator
```

3. Verify an IP address on the storage network has been assigned:

```
oc rsh <rook-ceph-operator-pod> ip a
```


### Create the `CephCluster` CR:

```bash
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-external
  namespace: rook-ceph
spec:
  external:
    enable: true
  network:
    provider: multus
    selectors:
      public: rook-ceph/storage
      cluster: rook-ceph/storage
  labelSelector: {}
```

```bash
oc create -f rook-external.yaml
```

### Test the Ceph setup

1. Verify the Ceph cluster status:

```bash
$ oc get CephCluster
NAME            DATADIRHOSTPATH   MONCOUNT   AGE     PHASE       MESSAGE                          HEALTH      EXTERNAL   FSID
rook-external                                5h56m   Connected   Cluster connected successfully   HEALTH_OK   true       <FSID>
```

**Note**:
The CR above is created by the `deploy.sh` script without the `network` section.
It is possible to copy it from the `EDPM` node and customize as described above.

```bash
scp -o StrictHostKeyChecking=no -i install_yamls/out/edpm/ansibleee-ssh-key-id_rsa \
    cloud-admin@192.168.122.100:/home/cloud-admin/rook-external.yaml .
```

2. Ensure the CSI RBD provisioner is running:


```bash
oc -n rook-ceph patch configmap rook-ceph-operator-config \
    --type='merge' -p '{"data": { "ROOK_CSI_ENABLE_RBD": "true" }}'
```

3. Check for all required Pods:

```bash
$ oc -n ceph-rook get pods

NAME                                         READY   STATUS    RESTARTS
csi-rbdplugin-6xmrh                          2/2     Running   0
csi-rbdplugin-provisioner-7d6d48cdb4-vr5ht   5/5     Running   0
rook-ceph-operator-56d9cdf7b4-xxkh9          1/1     Running   0
```

### Create a Test PVC - POD

1. Create a PVC using the `ceph-rbd` storageClass:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-rbd
```

2. Create a Pod using the PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer-one
spec:
  restartPolicy: Never
  containers:
   - image: gcr.io/google_containers/busybox
     command:
       - "/bin/sh"
       - "-c"
       - "while true; do echo $(hostname) $(date) >> /mnt/test/$(hostname); sleep 10; done"
     name: busybox
     volumeMounts:
       - name: mypvc
         mountPath: /mnt/test
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: myclaim
        readOnly: false
```

## Internal Ceph - ROOK / CRC

This guide explains how to set up an internal Ceph cluster using the ROOK operator in a CRC (CodeReady Containers) or SNO (Single Node OpenShift) environment. It assumes you're using the install_yamls project for development tools.

An internal Ceph cluster refers to a cluster deployed within OpenShift, with its lifecycle fully managed by the Rook operator. To ensure a functional deployment, a disk needs to be attached to the OpenShift (OCP) virtual machine.

### Steps:

1. Create and attach a disk to CRC with the following command:

```bash
make rook_crc_disk
```

2. Verify that the disk has been attached to the OCP instance:

```bash
oc debug node/$node -T -- chroot /host /usr/bin/bash -c lsblk
```

Replace `$node` with the result of the command `oc get nodes -o custom-columns=NAME:.metadata.name --no-headers`


3. Deploy the Rook operator:

```bash
make rook
```

4. Verify that the Rook operator is running:

```bash
oc project rook-ceph
oc get pods
```

5. Deploy the Ceph cluster:

```bash
make rook_deploy
```

6. Monitor the Ceph cluster deployment and check the progress:

```bash
oc get CephCluster -w
```

7. Wait until the cluster is ready and in a healthy state (`HEALTH_OK`).

8. Create a pool for the `RBD` `StorageClass` once the cluster is healthy:

```bash
ceph osd pool create replicapool 3
```

Set up the `StorageClass` and specify `replicapool` as the pool parameter:

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com # csi-provisioner-name
parameters:
  clusterID: rook-ceph # namespace:cluster
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/fstype: ext4
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

This guide helps you quickly set up and verify a basic internal Ceph cluster in your CRC or SNO environment.

## Additional Resources

- https://rook.io/docs/rook/latest-release/CRDs/Cluster/external-cluster/advance-external/#admin-privileges
- https://github.com/redhatci/ansible-collection-redhatci-ocp/blob/main/roles/odf_setup/tasks/openshift-storage-operator.yml#L13-L51
- https://github.com/rook/rook/issues/7563
