# Building and Pushing EDPM Bootc Images

This guide describes the current EDPM bootc host-image build source and a
site-specific way to include an Intel `ice` Dynamic Device Personalization
(DDP) profile. It also documents how to update nodes to a new bootc image.

## Build the EDPM bootc host image

The EDPM bootc host image is built in the
[`s2i-openstack-containers`](https://github.com/openstack-k8s-operators/s2i-openstack-containers)
repository. Its source Containerfile is
[`containers/edpm/edpm-bootc/Containerfile`](https://github.com/openstack-k8s-operators/s2i-openstack-containers/blob/main/containers/edpm/edpm-bootc/Containerfile).
From a clone of that repository, build and push the standard host image using
your own registry, namespace, and tag:

```bash
REGISTRY=quay.io NAMESPACE=myorg TAG=edpm-base \
    ./build.sh build edpm/edpm-bootc
REGISTRY=quay.io NAMESPACE=myorg TAG=edpm-base \
    ./build.sh push edpm/edpm-bootc
```

The current Containerfile installs EDPM host RPMs and removes
`dracut-config-generic`; it does not add a DDP firmware profile or regenerate
initramfs for DDP. `USER_PACKAGES` installs repository RPMs and is not a means
to copy an arbitrary `.pkg` artifact. For DDP, maintain a site-specific image
customization as described below. Do not download an old
`edpm-image-builder/bootc/Containerfile`: the host-image Containerfile is now
in `s2i-openstack-containers`.

For the separate bootc-image-builder workflow that converts the published host
image to a QCOW2 image and packages it for EDPM deployment, follow the current
[`edpm-image-builder/bootc/README.rst`](https://github.com/openstack-k8s-operators/edpm-image-builder/blob/main/bootc/README.rst).
For bare-metal deployment, use the customized host image as the input to that
QCOW2 workflow, then set the packaged QCOW2 image on the node set:

```yaml
kind: OpenStackDataPlaneNodeSet
spec:
  baremetalSetTemplate:
    osContainerImageUrl: <custom-repo>:<custom-tag>-qcow2
    osImage: edpm-bootc.qcow2
```

See the linked README for the current packaging and deployment steps.

## Build a site-specific image with a DDP profile

The Intel `ice` driver looks for `intel/ice/ddp/ice.pkg` in the firmware root
and loads DDP during device initialization. Profiles are supplied as `.pkg`
files. On bootc, do not use a runtime role to change the firmware link,
regenerate initramfs, or reload the driver. Include the vendor/NIC-supported
profile in the image, make the canonical `ddp/ice.pkg` path resolve to it, and
regenerate initramfs during the image build. Switch the node to the custom
image and reboot to activate it. A running PF cannot be updated with a new
profile until the driver is unloaded and reloaded; that is not a bootc runtime
operation supported by this workflow.

Use a `.pkg` artifact provided and named according to the NIC vendor's
instructions. This guide does not prescribe an RPM or artifact download URL.
The profile file belongs under
`/lib/firmware/intel/ice/<edpm_ddp_package>/`; the directory name must equal
the `edpm_ddp_package` value supplied to the role (for example, `ddp-comm`).
The selected filename must satisfy the role's `ice*.pkg*` pattern. The
canonical `/lib/firmware/intel/ice/ddp/ice.pkg` link must resolve to that
selected file.

First build and push the standard image above. Then create a site-owned build
context with the supplied profile artifact copied locally as `ddp.pkg` and a
site-maintained `Containerfile.ddp` based on that published image. Set the
following values to match the image and artifact supplied for your NIC:

```bash
export EDPM_BOOTC_BASE="quay.io/myorg/openstack-edpm-bootc:edpm-base"
export DDP_PROFILE="ddp-comm" # Must equal edpm_ddp_package at runtime.
export DDP_FILENAME="<vendor-supported-ice-package>.pkg"
export EDPM_BOOTC_DDP_IMAGE="quay.io/myorg/openstack-edpm-bootc:ddp-custom"
```

Use the following as a site-specific Containerfile template. The `COPY`
instruction copies the user-supplied artifact from the build context; it is
not fetched from an RPM repository or URL. Keep the filename and directory
values consistent with the vendor's requirements.

```Dockerfile
ARG EDPM_BOOTC_BASE
FROM ${EDPM_BOOTC_BASE}

ARG DDP_PROFILE
ARG DDP_FILENAME
COPY ddp.pkg /tmp/site-ddp.pkg

RUN set -eu; \
    test -n "${DDP_PROFILE}"; \
    test -n "${DDP_FILENAME}"; \
    case "${DDP_FILENAME}" in ice*.pkg*) ;; *) echo "DDP_FILENAME must match ice*.pkg*" >&2; exit 1 ;; esac; \
    install -D -m 0644 /tmp/site-ddp.pkg "/lib/firmware/intel/ice/${DDP_PROFILE}/${DDP_FILENAME}"; \
    mkdir -p /lib/firmware/intel/ice/ddp; \
    ln -sfn "../${DDP_PROFILE}/${DDP_FILENAME}" /lib/firmware/intel/ice/ddp/ice.pkg; \
    command -v dracut; \
    for modules_dir in /usr/lib/modules/*; do \
        test -d "${modules_dir}" || continue; \
        kver="${modules_dir##*/}"; \
        dracut --force --add-drivers ice --install /lib/firmware/intel/ice/ddp/ice.pkg "${modules_dir}/initramfs.img" "${kver}"; \
    done; \
    rm -f /tmp/site-ddp.pkg

RUN bootc container lint
```

Build and push the site image from the customized build context:

```bash
podman build \
    --build-arg EDPM_BOOTC_BASE="${EDPM_BOOTC_BASE}" \
    --build-arg DDP_PROFILE="${DDP_PROFILE}" \
    --build-arg DDP_FILENAME="${DDP_FILENAME}" \
    -f Containerfile.ddp \
    -t "${EDPM_BOOTC_DDP_IMAGE}" .
podman push "${EDPM_BOOTC_DDP_IMAGE}"
```

### Verify the built image

Verify the profile file, canonical link, and initramfs payload independently in
the image. This checks that the bytes at the canonical firmware path are also
present at that path in each installed kernel's initramfs. It does not prove
NIC compatibility, successful device initialization, or that a profile was
accepted by a particular adapter.

```bash
podman run --rm \
    --env DDP_PROFILE \
    --env DDP_FILENAME \
    --entrypoint /bin/bash "${EDPM_BOOTC_DDP_IMAGE}" -ec '
set -euo pipefail
profile_file="/lib/firmware/intel/ice/${DDP_PROFILE}/${DDP_FILENAME}"
canonical="/lib/firmware/intel/ice/ddp/ice.pkg"
test -f "${profile_file}"
test -L "${canonical}"
test "$(readlink -f "${canonical}")" = "$(readlink -f "${profile_file}")"
runtime_sha256=$(sha256sum "${canonical}" | cut -d " " -f 1)
found_kernel=0
for modules_dir in /usr/lib/modules/*; do
    test -d "${modules_dir}" || continue
    found_kernel=1
    initramfs="${modules_dir}/initramfs.img"
    test -s "${initramfs}"
    initramfs_sha256=$(lsinitrd -f "${canonical}" "${initramfs}" | sha256sum | cut -d " " -f 1)
    test "${runtime_sha256}" = "${initramfs_sha256}"
    printf "%s: DDP SHA-256 %s matches the initramfs\n" "${initramfs}" "${runtime_sha256}"
done
test "${found_kernel}" = 1
'
```

After verifying the image, update the node to use the pushed custom image and
reboot it. The custom image build and runtime role invocation are separate: the
build bakes in the profile and initramfs. In an `edpm-ansible` version that
implements the bootc validation contract, the role path is read-only and
validates the profile selection, canonical runtime link, and payload in the
currently booted kernel's initramfs. It does not build or modify the image, or
validate NIC compatibility or device readiness.

When invoking `select_kernel_ddp_package` on a bootc node, pass the same
`edpm_ddp_package` value used as `DDP_PROFILE` during the image build. For
example, with `DDP_PROFILE=ddp-comm`:

```bash
ansible-playbook playbooks/select_kernel_ddp_package.yml \
    --extra-vars edpm_ddp_package=ddp-comm
```

This example describes the validation-only bootc behavior; it does not apply a
runtime firmware change. Use an `edpm-ansible` version that implements this
bootc validation contract.

## Updating EDPM Bootc Nodes

For EDPM nodes deployed with bootc images, the update process differs from traditional RPM-based updates. This section explains how to update bootc-based EDPM nodes.

### Overview

EDPM bootc nodes use a container-based operating system that can be updated by switching to new container images and rebooting.

There are two update procedures, a single phase update and a two-phase update.
With both procedures, OVN is still update initially on its own. It's the
remainder of the update that is either a single phase update or a two-phase
update.

The single phase update process is described in [EDPM update procedures](edpm_update_overview.md). This procedure update the system and services simultaneously.

The two-phase procedure (Technology Preview) updates the remaining services and
system separately, as described in [new EDPM update
overview](new_edpm_update_overview.md).

Both of these procedures require special handling for bootc, and will be
described below.

#### Required Ansible Variable

For both procedures, when updating the OS (system) of bootc nodes, you must specify the
target bootc container image using the following ansible variable:

```yaml
edpm_update_system_bootc_os_container_image: "your-registry.example.com/edpm-bootc:updated-version"
```

See the examples below which show setting the variable on the
OpenStackDataPlaneDeployment.

### Single phase update procedure (system and services together)

After updating OVN, update the rest of the EDPM services create an
OpenStackDataPlaneDeployment in which runs the `update` service. This step is
further described in [Updating other services on the data
plane](https://openstack-k8s-operators.github.io/openstack-operator/dataplane/#proc_updating-the-data-plane-dataplane).

1. **Build Updated Bootc Image**: First, build and push an updated bootc
   container image with the desired changes using the procedures described
   earlier in this document.

2. Create `edpm-compute-bootc-update.yaml`

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneDeployment
metadata:
  name: edpm-compute-bootc-update
spec:
  nodeSets:
    - edpm-compute-bootc
  servicesOverride:
    - update
    - reboot-os # Optional. Will automatically reboot the nodes
  ansibleExtraVars:
    edpm_update_system_bootc_os_container_image: "your-registry.example.com/edpm-bootc:updated-version"
    edpm_reboot_strategy: force # Optional, required when using reboot-os with already deployed nodes
```

**Important:** When using the `reboot-os` service with `edpm_reboot_strategy: force`, reboots are immediate and disruptive. A forced reboot will terminate all running processes, which may cause workload interruption. There is no reverting from a forced reboot once initiated. Always plan forced reboots during scheduled maintenance windows and ensure all critical services are properly prepared for immediate shutdown. Please follow the instruction on [Rebooting Compute Nodes](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/updating_your_environment_to_the_latest_maintenance_release/proc_rebooting-compute-nodes_perform-update) to plan your reboot.

3. Apply `edpm-compute-bootc-update.yaml`

```bash
oc apply -f edpm-compute-bootc-update.yaml
```

4. Verify the OpenStackDataPlaneDeployment completes successfully.

5. Reboot the nodes to activate the new image if the `reboot-os` service was not
   included on the OpenStackDataPlaneDeployment.

### Two-phase update procedure (system then services separately)

#### Phase 1: OpenStack Services Update

The first phase updates OpenStack service containers and essential RPM packages. For bootc nodes, this phase has special considerations:

**Important for Bootc Nodes**: If you intend to run only `edpm_update_services`
(Phase 1) but there is a newer version of `openstack-selinux` available, you
**must first build and update to a new bootc image** that includes the updated
`openstack-selinux` package.

This is because bootc nodes cannot update individual RPM packages like
`openstack-selinux` at runtime - all system packages must be included in the
bootc container image. The high level process is:

##### openstack-selinux update

If necessary, perform the following steps to update only openstack-selinux on
bootc nodes.

1. **Check for openstack-selinux updates**: Verify if a newer version of `openstack-selinux` is available in your repositories

2. **Build updated bootc image**: If needed, rebuild your bootc image with the newer `openstack-selinux` version

3. Create `edpm-update-system-selinux.yaml`

```yaml
# Step 1: Update bootc image with newer openstack-selinux (if needed)
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneDeployment
metadata:
  name: edpm-update-system-selinux
spec:
  nodeSets:
    - edpm-compute-bootc
  servicesOverride:
    - update-system
    - reboot-os # Optional. Will automatically reboot the nodes
  ansibleExtraVars:
    edpm_update_system_bootc_os_container_image: "your-registry.example.com/edpm-bootc:updated-selinux"
    edpm_reboot_strategy: force # Optional, required when using reboot-os with already deployed nodes
```

**Important:** When using the `reboot-os` service with `edpm_reboot_strategy: force`, reboots are immediate and disruptive. A forced reboot will terminate all running processes, which may cause workload interruption. There is no reverting from a forced reboot once initiated. Always plan forced reboots during scheduled maintenance windows and ensure all critical services are properly prepared for immediate shutdown. Please follow the instruction on [Rebooting Compute Nodes](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/updating_your_environment_to_the_latest_maintenance_release/proc_rebooting-compute-nodes_perform-update) to plan your reboot.

4. Apply `edpm-update-system-selinux.yaml`

```bash
oc apply -f edpm-update-system-selinux.yaml
```

5. Reboot the nodes to activate the new image if the `reboot-os` service was not
   included on the OpenStackDataPlaneDeployment.

6. Create `edpm-update-services-bootc.yaml` Deployment to update the services.

```yaml
# Step 2: Update OpenStack services
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneDeployment
metadata:
  name: edpm-update-services-bootc
spec:
  nodeSets:
    - edpm-compute-bootc
  servicesOverride:
    - update-services
```

7. Apply `edpm-update-services-bootc.yaml`

```bash
oc apply -f edpm-update-services-bootc.yaml
```

8. Verify the OpenStackDataPlaneDeployment completes successfully.

No reboot is required since only the remaining services were updated.

#### Phase 2: System Update (Bootc Image Switch)

**Important**: For bootc nodes, the system update phase requires switching to a new bootc container image instead of updating individual RPM packages.

1. **Build Updated Bootc Image**: First, build and push an updated bootc container image with the desired changes using the procedures described earlier in this document.

2. Create `edpm-update-bootc.yaml` to update the bootc system, and specify the
   image built in the previous step.

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneDeployment
metadata:
  name: edpm-update-bootc
spec:
  nodeSets:
    - edpm-compute-bootc
  servicesOverride:
    - update-system
    - reboot-os # Optional. Will automatically reboot the nodes
  ansibleExtraVars:
    # Specify the new bootc image to switch to
    edpm_update_system_bootc_os_container_image: "your-registry.example.com/edpm-bootc:new-version"
    edpm_reboot_strategy: force # Optional, required when using reboot-os with already deployed nodes
```

**Important:** When using the `reboot-os` service with `edpm_reboot_strategy: force`, reboots are immediate and disruptive. A forced reboot will terminate all running processes, which may cause workload interruption. There is no reverting from a forced reboot once initiated. Always plan forced reboots during scheduled maintenance windows and ensure all critical services are properly prepared for immediate shutdown. Please follow the instruction on [Rebooting Compute Nodes](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/updating_your_environment_to_the_latest_maintenance_release/proc_rebooting-compute-nodes_perform-update) to plan your reboot.

3. Apply `edpm-update-bootc.yaml`

```bash
oc apply -f edpm-update-bootc.yaml
```

4. Verify the OpenStackDataPlaneDeployment completes successfully.

5. Reboot the nodes to activate the new image if the `reboot-os` service was not
   included on the OpenStackDataPlaneDeployment.

### Important Considerations

- **Reboot Requirement**: Unlike traditional RPM updates, bootc image switches always require a reboot to take effect
- **openstack-selinux Dependencies**: Always check if `edpm_update_services` requires a newer version of `openstack-selinux` before proceeding. If so, build and switch to an updated bootc image first
- **Maintenance Windows**: Schedule the system update phase during planned maintenance windows due to the reboot requirement
- **Image Availability**: Ensure the target bootc image is accessible from all EDPM nodes before starting the update
- **Package Updates**: Remember that bootc nodes cannot update individual RPM packages - all updates must be included in the bootc container image
- **Rollback**: Bootc supports rollback to previous images using `bootc rollback` if issues occur
- **Verification**: After reboot, verify the new image is active using `bootc status`

For more information on deploying and updating these images, refer to the OpenStack EDPM documentation and the update procedure guides linked above.
