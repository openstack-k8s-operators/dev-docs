# Testing OpenStack code changes in the local podified environment

You can test your changes made to the OpenStack project (Neutron will be used as
an example in this document) in the OpenStack deployed on top of the OpenShift,
like e.g. CRC. To do so, couple of steps are needed.
This guide is based on the similar guide from the
[edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/docs/source/testing_with_ansibleee.rst)
repository.

## Create local NFS storage

When using OpenShift Local (aka CRC), your export will be something like this:

```
% echo "${HOME}/dev/openstack/neutron/neutron 192.168.130.0/24(rw,sync,no_root_squash)" > /etc/exports

% exportfs -r
```

Make sure nfs-server and firewalld are started:
```
% systemctl start firewalld
% systemctl start nfs-server
```

Tip
---
CRC installs its own firewall rules, which likely will need to be adjusted
depending on the location of your NFS server. If your openstack project (e.g.
Neutron) directory is on the same system that hosts your CRC, then the simplest
thing to do is insert a rule that essentially circumvents the other rules:

```
% nft add rule inet firewalld filter_IN_libvirt_pre accept
```

## Create PV and PVC

Create an NFS PV and PVC which can be mounted on the POD with the OpenStack
service, like e.g. neutron-api.

```
NFS_SHARE=<Path to your directory with code>
NFS_SERVER=<IP of your NFS server>
cat <<EOF >openstack-code-storage.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: neutron-code
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadOnlyMany
  # IMPORTANT! The persistentVolumeReclaimPolicy must be "Retain" or else
  # your code will be deleted when the volume is reclaimed!
  persistentVolumeReclaimPolicy: Retain
  storageClassName: neutron-code
  mountOptions:
    - nfsvers=4.1
  nfs:
    path: ${NFS_SHARE}
    server: ${NFS_SERVER}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: neutron-code-dev
spec:
  storageClassName: neutron-code-dev
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
EOF

oc apply -f openstack-code-storage.yaml
```

## Add extraMount to Your OpenStackControlPlane CR

**Note**
This method will work only for the services which operators supports
`extraMounts` feature, like e.g. Neutron-operator.

New volume with your local code needs to be mounted in the POD with the service
which You are going to test, for example Neutron:

```
spec:
  neutron:
    template:
      extraMounts:
        - extraVol:
          - extraVolType: neutron-code
            mounts:
            - mountPath: /usr/lib/python3.9/site-packages/neutron
              name: neutron-code
            volumes:
            - name: neutron-code
              persistentVolumeClaim:
                claimName: neutron-code
                readOnly: true
          name: neutron-code
```

You can use kustomize or `oc edit` to update Your OpenStackControlPlane CR.
Once this will be done, POD(s) with your service will be recreated and You
should see new volume mounted there:

```
% oc describe $(oc get pods -l service=neutron -o name)
...
Status:
      Mounts:
      /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem from combined-ca-bundle (ro,path="tls-ca-bundle.pem")
      /usr/lib/python3.9/site-packages/neutron from neutron-code (rw)
      /var/lib/config-data/default from config (ro)
      /var/lib/config-data/tls/certs/ovndb.crt from ovndb-tls-certs (ro,path="tls.crt")
      /var/lib/config-data/tls/certs/ovndbca.crt from ovndb-tls-certs (ro,path="ca.crt")
      /var/lib/config-data/tls/private/ovndb.key from ovndb-tls-certs (ro,path="tls.key")
      /var/lib/kolla/config_files/config.json from config (ro,path="neutron-api-config.json")
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rhk5m (ro)
```
