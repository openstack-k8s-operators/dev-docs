# Overview of Configuring Services on EDPM Nodes

This document gives a high level overview of how to create a service
which needs to be hosted on a RHEL-based External Dataplane Management
(EDPM) node.

## OpenStack Operator

Though the service may not run directly on OpenShift workers, the
service definition must be made through the Custom Resources of the
[OpenStack Operator](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/#custom-resources)
which can run Ansible playbooks to configure the service.

The service should fit the
[composable service model](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/#_composable_services)
so that the deployer can orchestrate when and how the new service is deployed.

A PR sent to the
[openstack-operator repository](https://github.com/openstack-k8s-operators/openstack-operator)
for a new service should contain an `OpenStackDataPlaneService`
definition ([examples](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/services))
and a sample of how to use the service in an `OpenStackDataPlaneNodeSet`
([examples](https://github.com/openstack-k8s-operators/openstack-operator/tree/main/config/samples)).

See the
[OpenStack Operator documentation](https://openstack-k8s-operators.github.io/openstack-operator)
for more information.

## EDPM Ansible

The Ansible playbook(s) and role(s) which configure the new service
should be contributed to the
[edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible)
repository. The Ansible Execution Environment (AEE) pod, which is
created by the dataplane operator, contains a copy of this repository
in its container image.

See the
[EDPM Ansible documentation](https://openstack-k8s-operators.github.io/edpm-ansible)
for more information.

## EDPM Service Patterns

Common usage patterns of Data Plane Operator custom resources and
EDPM Ansible for services which run on EDPM nodes.

### Configuration Files

As described in
[Services configuration / k8s operators](edpm_service_overview.md),
`Secrets` and `ConfigMaps` can store configuration information. If an
[extraMount](https://github.com/fultonj/docs/blob/main/extra_mounts.md)
is used in an `OpenStackDataPlaneNodeSet`, then the AEE pod will mount
those `Secrets` and `ConfigMaps`. Because Ansible will then have
direct access to that data as files, the
[ansible.builtin.copy module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
(or similar) may be used to put that configuration data on an EDPM node.

### Firewall

If the new service requires firewall ports to be opened, then the
service's EDPM Ansible role should create a YAML file in
`/var/lib/edpm-config/firewall/` on EDPM nodes with the appropriate
syntax. For example, the following file in this directory opens the
ports for the Ceph Monitoring service.

```yaml
- rule_name: "110 allow ceph_mon"
  rule:
    proto: tcp
    dport: [6789, 3300]
```

When the `configure-os` and `run-os`
[composable services](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/#_composable_services/)
run, they execute the role
[edpm_nftables](https://github.com/openstack-k8s-operators/edpm-ansible/tree/main/roles/edpm_nftables)
This role reads files in `/var/lib/edpm-config/firewall/`
and creates a `edpm-rules.nft` file in `/etc/nftables/` and then
configures the live firewall to use it.

For example, the rule above results in the following lines in
`/etc/nftables/edpm-rules.nft`

```command
[root@edpm-compute-0 ~]# grep ceph /etc/nftables/edpm-rules.nft
# 110 allow ceph_mon {'proto': 'tcp', 'dport': [6789, 3300]}
add rule inet filter EDPM_INPUT tcp dport { 6789,3300 } ct state new counter accept comment "110 allow ceph_mon"
[root@edpm-compute-0 ~]#
```
which results in the following output from the NFT command.
```command
[root@edpm-compute-0 ~]# nft list ruleset | grep ceph_mon
		tcp dport { 3300, 6789 } ct state new counter packets 0 bytes 0 accept comment "110 allow ceph_mon"
[root@edpm-compute-0 ~]#
```
If the service needs to be deployed after the `configure-os` and
`run-os` services have run, then the Ansible for that service can
directly call the `edpm_nftables` role to update the files in
`/etc/nftables` and reload the rules. An example of this from the
`edpm_libvirt` role is below.

```yaml
- name: Copy qemu vnc firewall config
  ansible.builtin.template:
    src: "firewall.yaml"
    dest: "/var/lib/edpm-config/firewall/vnc.yaml"
    mode: "640"
- name: Configure firewall for the vnc
  ansible.builtin.include_role:
      name: osp.edpm.edpm_nftables
      tasks_from: "configure.yml"
- name: Reload firewall for new vnc rule
  ansible.builtin.include_role:
      name: osp.edpm.edpm_nftables
      tasks_from: "run.yml"
```
### NTP

The openstack-operator provided service `run-os` (described in
[composable services](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/#_composable_services))
calls an edpm-ansible role to configure the system clock to
synchronize with NTP servers. If the new service requires NTP,
then put it after `run-os` on the services list in the service's
sample `OpenStackDataPlaneNodeSet` file.
