# Building and Pushing EDPM Bootc Images

This guide provides step-by-step instructions for building and pushing EDPM
bootc container images and QCOW2 disk images to your own container registry.

## Prerequisites

### System Requirements

- RHEL 9.4+, CentOS Stream 9, or Fedora 41+ host system.
- sudo access for privileged container operations
- At least 10GB of free disk space
- Container registry access (push permissions)

### Required Packages

Install the following packages on your build host:

```bash
sudo dnf install -y podman buildah osbuild-selinux
```

## Environment Variables

Set these environment variables to customize your build. Replace
`your-registry.example.com` with your actual container registry where you have
push permissions.

```bash
# Container registry and image names (CUSTOMIZE THESE)
export EDPM_BOOTC_REPO="your-registry.example.com/edpm-bootc"
export EDPM_BOOTC_TAG="latest"
export EDPM_BOOTC_IMAGE="${EDPM_BOOTC_REPO}:${EDPM_BOOTC_TAG}"
export EDPM_QCOW2_IMAGE="${EDPM_BOOTC_REPO}:${EDPM_BOOTC_TAG}-qcow2"

# Build configuration
export EDPM_BASE_IMAGE="quay.io/centos-bootc/centos-bootc:stream9"
export EDPM_CONTAINERFILE="Containerfile"
export RHSM_SCRIPT="empty.sh"  # Use "rhsm.sh" for RHEL with subscription
export FIPS="1"  # Set to "0" to disable FIPS mode
export USER_PACKAGES=""  # Space-separated list of additional packages to install
```

## Step 1: Obtain initial build content

The files and content to build the bootc image can be obtained by either
cloning the edpm-image-builder, or downloading the files.

### Obtain initial build content by cloning the repository

Clone the edpm-image-builder repository:

```bash
git clone https://github.com/openstack-k8s-operators/edpm-image-builder.git
cd edpm-image-builder
```

### Obtain initial build content by downloading

Download the necessary files:

```bash
mkdir -p bootc
pushd bootc
curl -O https://raw.githubusercontent.com/openstack-k8s-operators/edpm-image-builder/55ba53cf215b14ed95bc80c8e8ed4b29a45fd4ae/bootc/Containerfile
curl -O https://raw.githubusercontent.com/openstack-k8s-operators/edpm-image-builder/refs/heads/main/bootc/rhsm.sh
chmod +x rhsm.sh
mkdir ansible-facts
pushd ansible-facts
curl -O https://raw.githubusercontent.com/openstack-k8s-operators/edpm-image-builder/55ba53cf215b14ed95bc80c8e8ed4b29a45fd4ae/bootc/ansible-facts/bootc.fact
popd
popd
curl -O https://raw.githubusercontent.com/openstack-k8s-operators/edpm-image-builder/55ba53cf215b14ed95bc80c8e8ed4b29a45fd4ae/Containerfile.image
curl -O https://raw.githubusercontent.com/openstack-k8s-operators/edpm-image-builder/55ba53cf215b14ed95bc80c8e8ed4b29a45fd4ae/copy_out.sh
chmod +x copy_out.sh
```

## Step 2: Prepare the Build Environment

Navigate to the bootc directory:

```bash
cd bootc/
```

Create the output directory:

```bash
mkdir -p output/yum.repos.d
```

## Step 3: Set Up Repository Configuration

### Centos Stream 9

#### Download and Install repo-setup Tool

```bash
pushd output
curl -sL https://github.com/openstack-k8s-operators/repo-setup/archive/refs/heads/main.tar.gz | tar xvz
pushd repo-setup-main
python3 -m venv ./venv
source ./venv/bin/activate
PBR_VERSION=0.0.0 python3 -m pip install ./
cp venv/bin/repo-setup ../repo-setup
popd
```

#### Generate Repository Configuration

Configure the OpenStack rpm repositories to use during the container image
build:

```bash
# Set repository configuration (optional, these are defaults)
export REPO_SETUP="current-podified"
export REPO_SETUP_BRANCH="antelope"
export REPO_SETUP_DISTRO="centos9"
export REPO_SETUP_MIRROR="https://trunk.rdoproject.org"

# Generate repository files
mkdir -p output/yum.repos.d
pushd output
repo-setup \
    --output-path yum.repos.d \
    --branch ${REPO_SETUP_BRANCH} \
    --rdo-mirror ${REPO_SETUP_MIRROR} \
    -d ${REPO_SETUP_DISTRO} \
    ${REPO_SETUP}
popd
```

### RHEL 9.4+

For RHEL-based builds with subscription-manager, modify `rhsm.sh`:

1. Edit the subscription variables in `rhsm.sh`:
   ```bash
   RHSM_USER=your_username
   RHSM_PASSWORD=your_password
   RHSM_POOL=your_pool_id  # Only if SCA is disabled
   ```

2. Further edit `rhsm.sh` as needed, such as to run a `subscription-manager`
   command with an activation key. Or, use any custom script to enable any
   needed repositories.

2. Use the RHEL subscription script:
   ```bash
   export RHSM_SCRIPT="rhsm.sh"
   ```

## Step 4: Build the Bootc Container Image

### Configure Base Image

You can use different base images:

> #### RHEL 9.4+
> ```bash
> # For RHEL 9 (requires subscription)
> export EDPM_BASE_IMAGE="registry.redhat.io/rhel9/rhel-bootc:9.4"
> ```
>
> Login to registry.redhat.io
> ```bash
> sudo podman login registry.redhat.io
> ```

> #### Centos Stream 9
> ```bash
> # For CentOS Stream 9 (default)
> export EDPM_BASE_IMAGE="quay.io/centos-bootc/centos-bootc:stream9"
> ```

Build the EDPM bootc container image:

```bash
sudo buildah bud \
    --build-arg EDPM_BASE_IMAGE=${EDPM_BASE_IMAGE} \
    --build-arg RHSM_SCRIPT=${RHSM_SCRIPT} \
    --build-arg FIPS=${FIPS} \
    --build-arg USER_PACKAGES="${USER_PACKAGES}" \
    --volume /etc/pki/ca-trust:/etc/pki/ca-trust:ro,Z \
    --volume $(pwd)/output/yum.repos.d:/etc/yum.repos.d:rw,Z \
    -f ${EDPM_CONTAINERFILE} \
    -t ${EDPM_BOOTC_IMAGE} \
    .
```

### Verify the Build

Check that the image was created successfully:

```bash
sudo podman images | grep ${EDPM_BOOTC_REPO}
```

## Step 5: Push the Bootc Container Image

Push the bootc container image to your registry:

```bash
sudo podman push ${EDPM_BOOTC_IMAGE}
```

## Step 6: Generate a QCOW2 Disk Image (Optional)

If you need a QCOW2 disk image for deployment:

> ### RHEL 9.4+
> ```bash
> export BUILDER_IMAGE="registry.redhat.io/rhel9/bootc-image-builder:latest"
> ```

> ### Centos Stream 9
> ```bash
> export BUILDER_IMAGE="quay.io/centos-bootc/bootc-image-builder:latest"
> ```

```bash
sudo podman run \
    --rm \
    -it \
    --privileged \
    --security-opt label=type:unconfined_t \
    -v ./output:/output \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    ${BUILDER_IMAGE} \
    --type qcow2 \
    --local \
    ${EDPM_BOOTC_IMAGE}
```

Move and checksum the generated QCOW2:

```bash
pushd output
sudo mv qcow2/disk.qcow2 edpm-bootc.qcow2
sudo sha256sum edpm-bootc.qcow2 > edpm-bootc.qcow2.sha256
popd
```

## Step 7: Package QCOW2 in Container (Optional)

To distribute the QCOW2 image as a container:

### Prepare packaging files:

```bash
cp ../copy_out.sh output/
cp ../Containerfile.image output/
```

### Build the QCOW2 container:

> #### RHEL 9.4+
>
> ```bash
> export BASE_IMAGE=registry.redhat.io/rhel9-4-els/rhel:9.4
> ```

> #### Centos Stream 9
>
> ```bash
> export BASE_IMAGE=quay.io/centos/centos:stream9-minimal
> ```

```bash
pushd output
sudo buildah bud \
    --build-arg IMAGE_NAME=edpm-bootc \
    --build-arg BASE_IMAGE=${BASE_IMAGE}
    -f ./Containerfile.image \
    -t ${EDPM_QCOW2_IMAGE} \
    .
popd
```

### Push the QCOW2 container:

```bash
sudo podman push ${EDPM_QCOW2_IMAGE}
```

## Customization Options

### FIPS Configuration

To disable FIPS mode:

```bash
export FIPS="0"
```

### Package Customization

The Containerfile defines several package groups that can be customized by
modifying the build args. Key package categories include:

- `BOOTSTRAP_PACKAGES`: Core system packages
- `OVS_PACKAGES`: Open vSwitch packages
- `PODMAN_PACKAGES`: Container runtime packages
- `LIBVIRT_PACKAGES`: Virtualization packages
- `CEPH_PACKAGES`: Ceph storage packages
- `USER_PACKAGES`: User-customizable packages for additional functionality

#### Adding Custom Packages with USER_PACKAGES

The `USER_PACKAGES` build argument allows you to inject additional packages into the bootc image during build time. This is useful for adding site-specific tools, drivers, or utilities that are not included in the default package sets.

**Usage:**

```bash
# Add custom packages during build
export USER_PACKAGES="vim htop strace tcpdump"

sudo buildah bud \
    --build-arg EDPM_BASE_IMAGE=${EDPM_BASE_IMAGE} \
    --build-arg RHSM_SCRIPT=${RHSM_SCRIPT} \
    --build-arg FIPS=${FIPS} \
    --build-arg USER_PACKAGES="${USER_PACKAGES}" \
    --volume /etc/pki/ca-trust:/etc/pki/ca-trust:ro,Z \
    --volume $(pwd)/output/yum.repos.d:/etc/yum.repos.d:rw,Z \
    -f ${EDPM_CONTAINERFILE} \
    -t ${EDPM_BOOTC_IMAGE} \
    .
```

**Examples:**

```bash
# Add debugging and monitoring tools
export USER_PACKAGES="vim htop strace tcpdump iperf3 curl wget"

# Add storage-related utilities
export USER_PACKAGES="lvm2 multipath-tools sg3_utils"

# Add network debugging tools
export USER_PACKAGES="nmap netcat-openbsd traceroute mtr"

# Add development tools
export USER_PACKAGES="git gcc make python3-pip"
```

**Important Notes:**

- Package names must be valid for the base OS repository (CentOS Stream 9 or RHEL 9)
- Packages must be available in the configured repositories
- Adding too many packages will increase the image size
- Verify package availability before building to avoid build failures

## Troubleshooting

### Build Fails with Permission Errors

Ensure you're using `sudo` with buildah and podman commands for privileged
operations.

### Repository Configuration Issues

If package installation fails, verify your repository configuration:

```bash
ls -la output/yum.repos.d/
```

### Container Registry Authentication

Ensure you're logged into your container registry:

```bash
sudo podman login your-registry.example.com
```

### Storage Space Issues

Bootc images can be large. Clean up old images if needed:

```bash
sudo podman system prune -a
```

## Image Contents

The resulting bootc image includes:

- **Base OS**: CentOS Stream 9 or RHEL 9 with bootc
- **OpenStack Components**: Packages for EDPM node functionality
- **Container Runtime**: Podman and Buildah for workload containers
- **Networking**: Open vSwitch and NetworkManager configuration
- **Storage**: LVM, multipath, and Ceph client tools
- **Virtualization**: Libvirt and QEMU for compute nodes
- **Security**: FIPS mode enabled by default
- **Monitoring**: System monitoring and logging tools

### Extract QCOW2 from Container (Usage)

To extract the QCOW2 image from the container:

```bash
# Create directory for extracted images
mkdir -p /path/to/extracted/images

# Extract QCOW2 from container
podman run \
    --volume /path/to/extracted/images:/target:Z \
    --rm \
    ${EDPM_QCOW2_IMAGE}
```

## Cleanup

To clean up build artifacts:

```bash
# Remove output directory
sudo rm -rf output

# Remove container images (optional)
sudo podman rmi --force ${EDPM_BOOTC_IMAGE} || true
sudo podman rmi --force ${BUILDER_IMAGE} || true
sudo podman rmi --force ${EDPM_QCOW2_IMAGE} || true
```

## Using the QCOW2 Container Image with OpenStackDataPlaneNodeSet

After building and pushing your QCOW2 container image, you can use it when creating an OpenStackDataPlaneNodeSet resource. The key fields to configure are `osImage` and `osContainerImageUrl`.

### Example OpenStackDataPlaneNodeSet Configuration

```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
metadata:
  name: edpm-compute-bootc
  namespace: openstack
spec:
  preProvisioned: False
  # Baremetal configuration with OS image settings
  baremetalSetTemplate:
    # Configure the OS image settings
    osImage: edpm-bootc.qcow2
    osContainerImageUrl: your-registry.example.com/edpm-bootc:latest-qcow2

```

### Field Descriptions

- **`baremetalSetTemplate.osImage`**: The filename of the QCOW2 image that will be extracted from the container. This should match the filename inside your QCOW2 container image (typically `edpm-bootc.qcow2`).

- **`baremetalSetTemplate.osContainerImageUrl`**: The full URL to your QCOW2 container image that was built and pushed in the previous steps. This is the image tagged with `-qcow2` suffix (e.g., `your-registry.example.com/edpm-bootc:latest-qcow2`).


### Verification

After applying the OpenStackDataPlaneNodeSet, verify that the configuration is correct:

```bash
# Check the NodeSet status
oc get openstackdataplanenodeset edpm-compute-bootc -o yaml

# Verify the deployment progress
oc get openstackdataplanedeployment

# Check the logs for image download and extraction
oc logs -f deployment/dataplane-operator-controller-manager -n openstack-operators
```

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
    - reboot # Optional. Will automatically reboot the nodes
  ansibleVars:
    edpm_update_system_bootc_os_container_image: "your-registry.example.com/edpm-bootc:updated-version"
```

3. Apply `edpm-compute-bootc-update.yaml`

```bash
oc apply -f edpm-compute-bootc-update.yaml
```

4. Verify the OpenStackDataPlaneDeployment completes successfully.

5. Reboot the nodes to activate the new image if the `reboot` service was not
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
    - reboot # Optional. Will automatically reboot the nodes
  ansibleVars:
    edpm_update_system_bootc_os_container_image: "your-registry.example.com/edpm-bootc:updated-selinux"
```

4. Apply `edpm-update-system-selinux.yaml`

```bash
oc apply -f edpm-update-system-selinux.yaml
```

5. Reboot the nodes to activate the new image if the `reboot` service was not
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
    - reboot # Optional. Will automatically reboot the nodes
  ansibleVars:
    # Specify the new bootc image to switch to
    edpm_update_system_bootc_os_container_image: "your-registry.example.com/edpm-bootc:new-version"
```

3. Apply `edpm-update-bootc.yaml`

```bash
oc apply -f edpm-update-bootc.yaml
```

4. Verify the OpenStackDataPlaneDeployment completes successfully.

5. Reboot the nodes to activate the new image if the `reboot` service was not
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
