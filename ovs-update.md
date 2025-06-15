# Open vSwitch update

This document describes implications of updating Open vSwitch in OpenStack
cluster. It also explains the current approach, and ways to improve.

## Problem

Open vSwitch is a critical component in the OpenStack control plane. It
maintains network data path for cloud workloads (e.g. VMs, routers). It is
important to avoid unnecessary data plane disruptions when updating Open
vSwitch.

On the other hand, it's important to be able to roll out updates to Open
vSwitch in a timely manner to address security vulnerabilities and performance
issues.

When updating Open vSwitch, the following concerns should be addressed:

- When updating Open vSwitch, the `ovs-vswitchd` process may have to be restarted.
  While the process is restarting, the data plane may be disrupted. For
  example, if the kernel datapath is used, a previously unknown flow may have
  to be upcalled to `vswitchd` to make a decision. If the process is not
  running at this moment, the packet may be dropped.

- Once Open vSwitch is restarted, it may start with a clean slate for its
  OpenFlow tables. This will result in invalidation of old datapath flows.
  Until the controller (e.g. `ovn-controller`) reprograms the datapath, the
  traffic may be dropped.

- Depending on the type of RPM update (see details below), the package may
  leave Open vSwitch service disabled, leaving the data path in a
  non-operational state. The update procedure should make sure that the service
  is left enabled after any package update.

## Scenarios

Open vSwitch is deployed in the OpenStack cluster in multiple roles and form
factors:

- As a backend for OpenShift cluster networking solution (OpenShift SDN or
  OVN-Kubernetes). This role is not managed by OpenStack deployment. *This
  document is not concerned with this scenario.*

- As a backend for OVN gateways managed by OVNController CR controller. This
  Open vSwitch instance is podified and runs in DaemonSet managed pods.

- As a backend for OVN controllers running on data plane nodes. This Open
  vSwitch instance is running on bare metal and is deployed from RPM packages
  using `edpm_ovs` role.

## Podified Open vSwitch for OVN gateways

Open vSwitch for OVN gateways is deployed as a DaemonSet separate from the
DaemonSet serving `ovn-controller`. This is done specifically to avoid the need
to restart `ovs-vswitchd` pods when updating OVN (and vice versa).

Even so, updating Open vSwitch in this scenario may still cause data plane
disruption.

A new Open vSwitch for podified OVN gateways is shipped as a new container
image. During the image update, the DaemonSet will be updated to use the new
image. The DaemonSet update will cause the old pods to be terminated and new
pods to be started. This will necessarily restart the `ovs-vswitchd` processes.

Right now, there is no provision to mitigate the effect of this restart.
Meaning, the new process will start with empty OpenFlow tables and will
invalidate datapath flows. It will wait for `ovn-controller` to reconnect and
reconfigure it via OpenFlow control socket. In the meantime, the traffic may be
disrupted.

### Proposal

To mitigate the effect of `ovs-vswitchd` restart, we can utilize
`other_config:flow-restore-wait` mechanism in Open vSwitch.
[This mechanism](https://github.com/openvswitch/ovs/blob/fbade819d2de1685c49e1deaf62d45049a5c2a27/vswitchd/vswitch.xml#L104)
allows Open vSwitch to restore the OpenFlow tables from a backup file after a
restart.

This mechanism is already utilized in the Open vSwitch RPM package when doing
`systemctl reload openvswitch`. Since systemd unit files are not used to manage
`ovs-vswitchd` in pods, we should implement this mechanism in shutdown and
startup scripts used for OVS DaemonSet.[^0]

The proposed implementation would work as follows:

- Before shutting down `ovs-vswitchd`,
  [`PreStop` hook](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
  for OVS DaemonSet pods would set `other_config:flow-restore-wait` to `true`
  in the OVSDB. This will indicate to the new `ovs-vswitchd` process that it
  should not flush the kernel datapath or connect to its controllers until
  saved flows are restored.

- In addition, the `PreStop` hook would save the OpenFlow tables to a file in
  the pod's volume. This can be done with `ovs-ofctl dump-flows` command.

- After the new `ovs-vswitchd` process starts in the new pod, the startup script will check
  for the presence of the saved flows file. If it exists, it will restore the
  flows using `ovs-ofctl` command. It will then remove the file.

- Once the flows are restored, the startup script will set
  `other_config:flow-restore-wait` back to `false` in the OVSDB. This will
  allow the new `ovs-vswitchd` process to connect to its controllers and start
  processing packets.

## Bare metal Open vSwitch for data plane nodes

In contrast to podified Open vSwitch for OVN gateways, Open vSwitch on data
plane nodes is deployed on bare metal using RPM packages. The deployment is
managed by `edpm_ovs` role.

### RPM packaging

There are several important details to consider when updating Open vSwitch RPM
packages:

- In addition to the `openvswitch*` package, OpenStack also ships a
  `<distro>-openvswitch` wrapper package. (`rdo-openvswitch` for RDO,
  `rhoso-openvswitch` for RHOSO). This package is used to select the Open
  vSwitch "stream" to use (e.g. `3.0` vs `3.3`). It is needed because all Open
  vSwitch packages are shipped in the same repository.[^1]

- The `openvswitch*` package is actually a set of packages with different
  names, each unique name representing one of the major version "streams".
  E.g., `openvswitch3.0`, `openvswitch3.3`. This means that when updating from
  `3.0` to `3.3`, the `openvswitch3.0` package will be removed and a new
  package `openvswitch3.3` will be installed. RPM will trigger `%preun` script
  for the old `openvswitch3.0` package. This script will then handle the
  invocation as a complete package removal, including disabling the
  `openvswitch` service with `systemctl disable` command.[^2]

- On the other hand, the new `openvswitch3.3` package will be installed without
  the `openvswitch` service enabled.[^3]

The effect of the above is that after updating Open vSwitch from one major
"stream" to another (e.g. `3.0` to `3.3`), the `openvswitch` service will be
left disabled.

This is in contrast to a minor version update in the same major "stream". In
case when the new package version is in the same "stream", the update is
handled as a regular package update. It installs the new unit files for the
service, then `systemctl daemon-reload` is called to reload the systemd
configuration. Finally, `systemctl reload-or-restart openvswitch` is executed.
This in turn will gracefully reload the `ovs-vswitchd` process, using
`other_config:flow-restore-wait` mechanism to avoid data plane disruption.

> Note: the `other_config:flow-restore-wait` mechanism is not used when
> `systemctl restart openvswitch` is called. We should be careful to avoid
> using `restart`, if possible.

In the past, TripleO
[worked around](https://review.opendev.org/c/openstack/tripleo-ansible/+/742968/10/tripleo_ansible/ansible_plugins/modules/tripleo_ovs_upgrade.py#139)
this issue by avoiding execution of `%preun` scriptlets from the `openvswitch*`
packages. This was done by direct manipulation of `rpm` command line arguments.
This workaround is not currently implemented in this project. The rationale
behind the workaround back then was two-fold:

- To avoid disabling the `openvswitch` service on major Open vSwitch version
  updates, which would leave the data plane in a non-operational state.

- To avoid the need to restart the `ovs-vswitchd` process, which would cause
  interruption in the data plane. This was more important because back then,
  the `openvswitch` RPM package was not using `other_config:flow-restore-wait`
  mechanism for `systemctl reload`. This is no longer an issue.

### DPDK

When using userspace data plane (DPDK), the situation is more complex. In
contrast to the kernel data path, the DPDK data path is maintained by
[PMD threads](https://docs.openvswitch.org/en/latest/topics/dpdk/pmd/)
that are part of the `ovs-vswitchd` process. Their lifetime is tied to the
lifetime of the main `ovs-vswitchd` process. When the process is restarted, the
PMD threads are also restarted. In the meantime, no packets from the NIC are
delivered to the DPDK workload.

This scenario is more impactful because, in contrast, when the kernel data path
is used, the packet processing component is in the kernel and is not restarted
when `ovs-vswitchd` process is restarted. In the kernel data path scenario,
only unknown flows are upcalled and may be affected.

Hence, for DPDK scenario, it's more important to avoid restarting the
`ovs-vswitchd`, if possible.

### Proposal

> Note: The following proposal is based on the assumption that we *have* to
> maintain data plane intact during the update. An alternative approach to deal
> with the effect of the problem would be to accept the data plane disruption
> is inevitable and to never update the Open vSwitch package with workloads
> (VMs, routers) present on a node. While this may be our official guideline,
> it is still beneficial to minimize disruptions for when the workloads are
> present, for whatever reason. The proposal below flows from this attitude.

Considering the above, the following steps should be taken on the data plane
nodes to avoid unnecessary Open vSwitch data plane disruption:

- When updating Open vSwitch from one major version to another, the
  `openvswitch` service should be re-enabled after the package update. (We may
  avoid distinguishing between major and minor version updates and *always*
  enable it after any Open vSwitch package update.) This can be achieved by
  installing a
  [`systemd` `preset` file](https://www.freedesktop.org/software/systemd/man/latest/systemd.preset.html)
  that will enable the `openvswitch` service after the new `openvswitch`
  package installation. The preset file can be packaged with the
  `<distro>-openvswitch` package.[^4]

- When updating Open vSwitch from one major version to another, we should also
  disable `%preun` scriptlet to avoid unnecessary disabling of the
  `openvswitch` service. This can be done by passing `--nopreun` to `rpm`
  command line when updating the package. This is the same approach that was
  used in TripleO in the past.

- When updating Open vSwitch within the same major version, the `openvswitch`
  service should be always restarted using `systemctl reload openvswitch`
  instead of `systemctl restart openvswitch` (if at all!).[^5]

- To avoid the need to restart the `ovs-vswitchd` process during RPM package
  update, we should disable `%post` scriptlets in the `openvswitch*` packages.
  This can be done by passing `--nopost` to `rpm` command line when updating
  the package. This is the same approach that was used in TripleO in the
  past.[^6][^7]

Last but not least,

- In general, we should discourage Open vSwitch package update during the
  runtime of the workloads. Update documentation should reflect this and the
  potential impact if done otherwise.

### Dismissed alternatives

One alternative to the above proposal would be to never update Open vSwitch,
either including all other packages, or the Open vSwitch package only, when
workloads are present on a node.

This approach may be helpful as a matter of policy or in particular workflows.
For example, during the
[adoption](https://github.com/openstack-k8s-operators/data-plane-adoption) of a
non-podified OpenStack cluster that happens with the workloads in-place, it may
be useful to separate the Open vSwitch package update into a separate
maintenance window, to reduce the risk surface of the adoption and to avoid
one-off implementation of OS package updates with in-place workloads.

Still, at some point in time, the Open vSwitch package will have to be updated.
The proposal above is designed to maximize non-disruptive options available to
deployers.

To avoid Open vSwitch package update, one could just avoid running
[`edpm_update`](https://github.com/openstack-k8s-operators/edpm-ansible/pull/623)
service in their deployment.

Alternatively, we could build a new mechanism to `versionlock` particular
packages using the identically named `dnf`
[plugin](https://dnf-plugins-core.readthedocs.io/en/latest/versionlock.html).
The mechanism would allow us to lock the Open vSwitch package to a particular
version, while updating the rest of the packages as needed. The mechanism would
have to also deal with unlocking the package version and updating it e.g. when
the node is ready for reboot to take advantage of a new kernel or other
maintenance activities.

[^0]: For reference, the `ovs-ctl` implementation of the reload mechanism can
    be found
    [here](https://github.com/openvswitch/ovs/blob/fbade819d2de1685c49e1deaf62d45049a5c2a27/utilities/ovs-lib.in#L679).

[^1]: The single repository is an answer to the requirement from different
    "layered" projects, including OpenStack and OpenShift, to be able to pick a
    particular Open vSwitch "stream" and stick to it.

[^2]: This behavior is needed for legit package removal scenario.

[^3]: This behavior is in line with the Fedora
    [policy](https://docs.fedoraproject.org/en-US/packaging-guidelines/DefaultServices/#_enabling_services_by_default)
    to leave most installed services disabled by default.

[^4]: Alternatively, the `<distro>-openvswitch` package could also execute
    `systemctl enable openvswitch` in its `%post` scriptlet.

[^5]: This is already the default for the RPM package. We may have to
    reimplement it if we e.g. skip all RPM scriptlets and not just `%preun`.

[^6]: Alternatively, we can use `dnf` with `--setopt=tsflags=noscripts` option.
    But note that `dnf` doesn't support disabling only some scriptlets, leaving
    the rest executed.

[^7]: Neither `rpm` nor `dnf` Ansible modules support these parameters, so we
    would have to use `command` module instead to directly execute the tools
    with the necessary command line options.
