# Tech Preview: New procedure of updating External Dataplane Management (EDPM) nodes

**Introduced in:** FR3

---

**WARNING:** This feature is in **Technology Preview**. It is not recommended for production use, and its API and behavior may change in future releases.

---

## Overview

As explained in [current fully supported EDPM update procedure](edpm_update_overview.md), the EDPM update process is executed in two separate steps of updating OVN, then updating the rest of the services.

This document explains a new procedure for updating the rest of the EDPM services. The update of the rest of the EDPM services in this feature is split into two phases:

- **Phase 1:** Update of OpenStack service containers and the associated system-level RPM packages they depend on.
- **Phase 2:** Update of all other system-related RPM packages.

With two-phased approach operators gain finer control over the update process, reducing risks and simplifying troubleshooting in the event of issues.

## Phase 1: Update OpenStack Services and RPM Dependencies

### Prerequisites

Before proceeding, ensure the following conditions are met:
- All OVN services on the EDPM nodes have been successfully updated as per the OpenStack minor update [procedure](version_updates.md#ordering-of-service-updates-normal-vs-minor-updates).

### Description

This phase focuses on updating only the required RPM packages, pulling new container images for the OpenStack services, and for some services re-rendering config files. It does not perform a full system-wide package update.

### Execution

1. Update of essential packages happens first. RPM package that is always included in essential packages list is `openstack-selinux`. This list of essential packages may change in the future. In this step we perform targeted packages update and assure that `kernel`, `kernel-core` and `openvswitch` are always excluded from update packages list.

2. Update containers part of the `edpm_update_services` role, which runs the necessary tasks to update running containerized services on EDPM nodes. That means typically, restarting containers with updated container image, but there are also some exceptions:

    - For services with just `run.yml` tasks files included, only new container image replaces old container image.

    - For service with both `update.yml` and `install.yml`(currently edpm_nova and telemetry services) there are additional steps of re-rendering config files of running Openstack services followed by new container image start.

After successful execution of above steps in a form of `OpenstackDataplaneDeployment`, `openstack-operator` updates `OpenstackVersion` CR to mark completion of OpenStack minor update sequence.

### Implementation Details

Implementation details:
-  ansible role can be found in [edpm_update_services](https://github.com/openstack-k8s-operators/edpm-ansible/tree/main/roles/edpm_update_services) of edpm-ansible repo.

- code for integration between openstack-operator and the role was introduce in this PR: https://github.com/openstack-k8s-operators/openstack-operator/pull/1478. 
It is worth to note that `EDPMServiceType` expected by `openstack-operator` to differentiate regular `OpenstackDataplaneDeployment`, from update type of deployment, must be either "update" (old procedure) or "update-services" (this procedure) - [**link**](https://github.com/openstack-k8s-operators/openstack-operator/blob/c1cdd8ccd743462311265ae0533d173162fd0f70/controllers/dataplane/openstackdataplanenodeset_controller.go#L564).

## Phase 2: Update System-Related RPM Packages

### Description

This phase performs a full system-wide update of all remaining RPM packages on the EDPM nodes. A key principle of this phase is its independence. It can be executed at any time, such as during a maintenance window, long after Phase 1 is complete, or without Phase 1 having been run at all. This allows applying critical system patches on EDPM nodes, without running full OpenStack update.

**Important:**
This phase is optional. You can use an alternative mechanism to update system packages, but you must follow the same execution order described here to avoid dataplane disruption. This step may require a node reboot (e.g., for a new kernel) and should be scheduled during a planned maintenance window.

This phase is also split into two major parts. The first part involves the Open vSwitch update, and the second one involves updating all other system-wide packages.

### Execution

1. Open vSwitch update logic is implemented directly in edpm_ovs role. `update.yaml` task file from the role is imported into edpm_update_system role.
 Current approach and implications of updating Open vSwitch in OpenStack on EDPM are explained in [ovs-update documentation.](ovs-update.md#bare-metal-open-vswitch-for-data-plane-nodes)

2. All other packages are updated next, except `openvswitch`. There is a way to exclude packages from update, with `edpm_update_system_exclude_packages` list if needed.

### Implementation Details

Implementation details can be found in [edpm_update_system](https://github.com/openstack-k8s-operators/edpm-ansible/tree/main/roles/edpm_update_system) role of edpm-ansible repo.
