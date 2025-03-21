# Overview of Updating Services on EDPM Nodes

This document gives a high level overview of updating services on RHEL-based
External Dataplane Management (EDPM) nodes.

## The update service

The EDPM update process is typically executed in two separate steps of updating
ovn, then updating the rest of the services. For more details on this process
see [Updating the data
plane](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/#assembly_updating-the-data-plane).

After updating ovn, the rest of the EDPM services are updated by the 
[update](https://https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/services/dataplane_v1beta1_openstackdataplaneservice_update.yaml)
[OpenStackDataPlaneService](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/#openstackdataplaneservice).

This service executes a single playbook,
[update](https://openstack-k8s-operators.github.io/edpm-ansible/playbooks/update.html),
which includes a single role,
[`edpm_update`](https://openstack-k8s-operators.github.io/edpm-ansible/roles/role-edpm_update.html).

The
[`edpm_update`](https://openstack-k8s-operators.github.io/edpm-ansible/roles/role-edpm_update.html)
role does **not** run all tasks that are run on an initial deployment. It only
runs the tasks necessary to update the services it manages.  Typically, this
means just updating packages and restarting containers with updated images, but
it depends on the other roles and how they are included. For a container managed
service this usually means that just the `run.yml` tasks file from that
service's role is included.

Care should be taken to include any other needed tasks on update. If new tasks
are added to a role, and those tasks are required, they may not be automatically
included on update if those tasks are outside of what is already included (such
as the `run.yml` tasks file).

In such cases, the included roles, and not the `edpm_update` role, should
include these new required tasks. For example, the required tasks should also be
added to the role's `run.yml` tasks file so that they are automatically included
by `edpm_update`. Or, the role could add a new tasks file just for update,
`update.yml`, that includes the necessary required tasks, and `edpm_update`
would then included `update.yml`.

When it can be assumed that required tasks have been run on update (such as when
new releases are branched), they should be dropped from being included on
update. This keeps the set of tasks that actually need to be run on update as
minimal as possible.