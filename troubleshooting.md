# Troubleshooting guide

## OpenStackControlPlane is not Ready

If the status of the Ready condition of OpenStackControlPlane CR is not True
then the control plane not deployed correctly and you will see a message
indicating which part of the control plane is the source of the problem.

\# TODO: change the example to show a real error
```sh
oc get -n openstack OpenStackControlPlane \
  -o jsonpath="{.items[0].status.conditions[?(@.type=='Ready')]}" \
  | jq
{
  "lastTransitionTime": "2024-04-05T07:56:53Z",
  "message": "Setup complete",
  "reason": "Ready",
  "status": "True",
  "type": "Ready"
}
```

Alternatively you can check all the conditions that are not reporting True
status.

```sh
oc get -n openstack OpenStackControlPlane \
  -o jsonpath="{range .items[0].status.conditions[?(@.status!='True')]}{.type} is {.status} due to {.message}{'\n'}{end}"
```

If you identified a specific service causing the issue then you can check the
status of the related CR. E.g. If the OpenStackControlPlane reports False for
OpenStackControlPlaneNovaReady condition then you need to check the status
of the Nova CR.

```sh
oc get -n openstack Nova/nova \
  -o jsonpath="{range .status.conditions[?(@.status!='True')]}{.type} is {.status} due to {.message}{'\n'}{end}"
```

You can also check the status of the operators
```sh
oc get pods -n openstack-operators -lopenstack.org/operator-name
NAME                                                              READY   STATUS    RESTARTS   AGE
barbican-operator-controller-manager-845b85cc46-lnxqp             2/2     Running   0          2d
cinder-operator-controller-manager-74c9cf8947-2v25m               2/2     Running   0          2d
...
```
the logs of the operators
```sh
oc logs -n openstack-operators -lopenstack.org/operator-name=nova
```
and the status and logs of the deployed openstack services
```sh
oc get pods -n openstack
```
```sh
oc logs -n openstack nova-api-0
```


## OpenStackDataPlaneDeployment is not Ready

You can check the status of the OpenStackDataPlaneDeployment and NodeSet CRs
```sh
oc get -n openstack OpenStackDataPlaneDeployment
NAME                   NODESETS                  STATUS   MESSAGE
edpm-deployment-ipam   ["openstack-edpm-ipam"]   False    Deployment error occurred nodeSet: openstack-edpm-ipam error: execution.name telemetry-edpm-deployment-ipam-openstack-edpm-ipam Execution.namespace openstack Execution.status.jobstatus: Failed
```
```sh
oc get -n openstack OpenStackDataPlaneNodeSet
NAME                  STATUS   MESSAGE
openstack-edpm-ipam   False    Deployment error occurred check deploymentStatuses for more details

```
The output will tell you which ansible executing job is failed, then you can
check the logs of that job.
```sh
oc logs -n openstack job/telemetry-edpm-deployment-ipam-openstack-edpm-ipam
```

You can also use `oc debug` to debug the failing job and interactively execute
ansible.
```sh
oc debug --as-root --keep-annotations job/telemetry-edpm-deployment-ipam-openstack-edpm-ipam
```
Once in the debug environment, `edpm-entrypoint` must be used to execute
`ansible-runner`. The `ansible-runner` working directory is `/runner`. The
`edpm-ansible` ansible collection is installed at
`/usr/share/ansible/collections/ansible_collections/osp/edpm`.
```sh
/bin/edpm-entrypoint ansible-runner run /runner -p osp.edpm.telemetry -i telemetry-edpm-deployment-ipam-openstack-edpm-ipam
```

If needed, changes can be made to affect the ansible execution for debugging
purposes. The following shows an example of modifying the inventory.
```sh
cp /runner/inventory/hosts /root/inventory
microdnf -y install vim
# modify inventory
# vim /root/inventory
# use modified inventory
/bin/edpm-entrypoint ansible-runner run /runner -p osp.edpm.telemetry -i telemetry-edpm-deployment-ipam-openstack-edpm-ipam --inventory /root/inventory
```

## Crashing Pods

Certain problems in the deployment can cause Pods to crash and terminate
repeatedly and ending up in `CrashLoopBackOff` state. In this state it is not
possible to directly log into the Pod via `oc rsh` to troubleshoot the issue.
However you can use `oc debug` to duplicate the crashing Pod without the
container command, probes, and labels and get a shell into it. If you need to
route traffic to the Pod then you can use `--keep-labels=true` in the
`oc debug` command to keep the duplicated Pod as a backed of the related
Service.

The debug command can be used not only on a specific `Pod` (`oc debug
cinder-api-0`) but also on any resource that creates pods such as
`Deployments`, `StatefulSets`, etc. as well as `Nodes`.  For example: `oc debug
StatefulSet/cinder-scheduler`.

By default `oc debug` will run `/bin/sh` on the first container of the pod, but
we can easily specify a different container to debug.  This is useful for pods
like the API where the first container is tailing a file and its the second
container the one we usually want to debug.  For example: `oc debug -c
cinder-api pod/cinder-api-0`.

We can use these last 2 functionalities, debugging non `Pod` resources and
specifying the container, to debug Init Containers.  For example: `oc debug
-c init Deployment/dnsmasq-dns`.

Please check the `oc debug --help` output for other available functionality
such as `--as-root`, `--as-user`, `--to-namespace`, `--node-name`, etc.

## Nova, Cinder or Glance unable to write to Ceph RBD Backend

If Nova, Cinder or Glance have been configured as described in the
[Ceph integration documentation](ceph.md) but you are not able to
write to the storage backend from these services then follow this
troubleshooting guide.

This guide will provide an example of troubleshooting a scenario where
an image is not written to Ceph in Glance. However, the steps are also
applicable Cinder or Nova being unable to write to Ceph. We assume
that the pods is not crashlooping as described in the section above.

1. Confirm the pod has the Ceph configuration files

The [Ceph integration documentation](ceph.md) describes using
`ExtraMounts` to place files like `ceph.client.openstack.keyring`
and `ceph.conf` in `/etc/ceph` in each pod which needs to connect
to Ceph. Connect to the pod with `oc rsh` and ensure that both of
these files are present.

If the Ceph files are missing, check the syntax of
the `ExtraMounts` section of the spec in the `OpenStackControlPlane`.

2. Confirm the pod can reach the Ceph cluster

The Ceph cluster should be reachable from the Storage network as
described in the [networking document](networking.md). The IPs and
ports of the Ceph cluster's monitors should be in
`/etc/ceph.conf`. From within the pod, read this and try to ping the
IP as well as connect to the port.
```
$ oc rsh -c glance-api glance-default-external-api-0
bash-5.1$ cat /etc/ceph/ceph.conf
# minimal ceph.conf for 7fbaf2fd-f70e-5625-b00e-65b33ee823a6
[global]
	fsid = 7fbaf2fd-f70e-5625-b00e-65b33ee823a6
	mon_host = [v2:172.18.0.101:3300/0,v1:172.18.0.101:6789/0] [v2:172.18.0.102:3300/0,v1:172.18.0.102:6789/0] [v2:172.18.0.100:3300/0,v1:172.18.0.100:6789/0]
bash-5.1$ nc -z -v 172.18.0.101 3300
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Connected to 172.18.0.101:3300.
Ncat: 0 bytes sent, 0 bytes received in 0.62 seconds.
bash-5.1$
```
If you are unable to connect, then troubleshoot the network.

3. Confirm you can read from the Ceph cluster

If the Ceph cluster is reachable via the network, then try to use the
`rbd` command to read from it. The `rbd` command is in Nova pods but
not Glance or Cinder pods. The Nova containers which use the cephx key
are hosted on Compute nodes, not OpenShift, and thus the NovaPods on
OpenShift do not have access to the ceph files describe in step 1.

First update the `extraMounts` option described in the
[Ceph integration documentation](ceph.md) so that the propagation
list includes the NovaAPI service.

```
spec:
  ...
  extraMounts:
    - name: v1
      region: r1
      extraVol:
        - propagation:
          - CinderVolume
          - GlanceAPI
          - NovaAPI
```
The above update will restart the NovaAPI pod. Connect to the
restarted pod using `oc rsh` and confirm the required keys
are present as described in step 1.

Once connected to the NovaAPI pod with `oc rsh` examine the cephx key.

```
bash-5.1$ cat /etc/ceph/ceph.client.openstack.keyring
[client.openstack]
   key = "<redacted>"
   caps mgr = allow *
   caps mon = profile rbd
   caps osd = profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=backups, profile rbd pool=images
bash-5.1$
```
Observe a pool name from the OSD caps, e.g. the `images` pool is in
the list of OSD caps. Try to list the contexts of that pool with the
RBD command.
```
/usr/bin/rbd --conf /etc/ceph/ceph.conf \
             --keyring /etc/ceph/ceph.client.openstack.keyring
             --cluster ceph --id openstack
             ls -l -p images | wc -l
```
If the above command returns an integer of 0 or greater, then the
cephx key provides adequate permission to reach from the Ceph cluster.

If the above command hangs but connectivity has been confirmed then
work with your Ceph administrator to get a working cephx keyring.
It's also possible that there is an MTU mismatch on the storage
network. If your are jumbo frames (MTU=9000), then all network
switch ports between servers using the interface with the MTU
must be updated to support jumbo frames. If this change
is not made on the switch, then problems may manifest on the
Ceph application layer. Verify that all hosts using the network using
jumbo frames can communicate at the desired MTU with a command
like `ping -M do -s 8972 172.18.0.101`. Do not test this command
if you are using the standard MTU (1500).

4. Confirm the Ceph cluster is able to write data

It's possible to be able to read data from a Ceph cluster but not be
able to write it, even if write permission has been granted in the
cephx keyring. Usually this indicates that the ceph cluster is
overloaded and unable to write new data.

Try to write some test data to the images pool using the commands
below.
```
# DATA=$(date | md5sum | cut -c-12)
# POOL=images
# RBD="/usr/bin/rbd --conf /etc/ceph/ceph.conf --keyring /etc/ceph/ceph.client.openstack.keyring --cluster ceph --id openstack"
# $RBD create --size 1024 $POOL/$DATA
^C
#
```
If the above had succeeded, then the test data could be deleted with
`$RBD rm $POOL/$DATA`.

In the example above the `rbd` command hung and was canceled. It was
then confirmed that the ceph cluster itself was reporting slow writes
on some of its OSDs. Because the ceph cluster did not have enough
computational resources to write new data the problem had to be
resolved on the ceph cluster itself and there was nothing wrong with
the client configuration.
