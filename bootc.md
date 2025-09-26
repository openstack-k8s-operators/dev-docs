# Building and Pushing EDPM Bootc Images

This guide provides step-by-step instructions for building and pushing EDPM
bootc container images and QCOW2 disk images to your own container registry.

## Prerequisites

### System Requirements

- RHEL 9.2+, CentOS Stream 9, or Fedora 41+ host system.
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
export BUILDER_IMAGE="quay.io/centos-bootc/bootc-image-builder:latest"
export EDPM_CONTAINERFILE="Containerfile"
export RHSM_SCRIPT="empty.sh"  # Use "rhsm.sh" for RHEL with subscription
export FIPS="1"  # Set to "0" to disable FIPS mode
export USER_PACKAGES=""  # Space-separated list of additional packages to install
```

## Step 1: Clone the Repository

Clone the edpm-image-builder repository:

```bash
git clone https://github.com/openstack-k8s-operators/edpm-image-builder.git
cd edpm-image-builder
```

## Step 2: Prepare the Build Environment

Navigate to the bootc directory:

```bash
cd bootc/
```

Create the output directory:

```bash
mkdir -p output
```

## Step 3: Set Up Repository Configuration

### Download and Install repo-setup Tool

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

### Generate Repository Configuration

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

## Step 4: Build the Bootc Container Image

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

## Step 6: Generate QCOW2 Disk Image (Optional)

If you need a QCOW2 disk image for bare metal deployment:

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

```bash
pushd output
sudo buildah bud \
    --build-arg IMAGE_NAME=edpm-bootc \
    -f ./Containerfile.image \
    -t ${EDPM_QCOW2_IMAGE} \
    .
popd
```

### Push the QCOW2 container:

```bash
sudo podman push ${EDPM_QCOW2_IMAGE}
```

## Step 8: Extract QCOW2 from Container (Usage)

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

## Customization Options

### RHEL Subscription Management

For RHEL-based builds with subscription, modify `rhsm.sh`:

1. Edit the subscription variables in `rhsm.sh`:
   ```bash
   RHSM_USER=your_username
   RHSM_PASSWORD=your_password
   RHSM_POOL=your_pool_id  # Only if SCA is disabled
   ```

2. Use the RHEL subscription script:
   ```bash
   export RHSM_SCRIPT="rhsm.sh"
   ```

### Base Image Options

You can use different base images:

```bash
# For RHEL 9 (requires subscription)
export EDPM_BASE_IMAGE="registry.redhat.io/rhel9/rhel-bootc:latest"

# For CentOS Stream 9 (default)
export EDPM_BASE_IMAGE="quay.io/centos-bootc/centos-bootc:stream9"
```

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

## Next Steps

After building and pushing your images:

1. **For Bare Metal**: Use the QCOW2 image for traditional bare metal
   deployment through OpenStackDataPlaneNodeSet
2. **Integration**: Configure your OpenStack deployment to use your custom
   images as shown above

For more information on deploying these images, refer to the OpenStack EDPM
documentation.
