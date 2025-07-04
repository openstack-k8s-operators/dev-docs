## Building custom OpenStack images


This document outlines the hotfix process for an OpenStack service deployed on
OpenShift. The procedure assumes the following:

* **Internal Registry Usage**: Use the OpenShift internal container image
  registry instead of a third-party registry. To enable access from EDPM nodes,
  expose the registry via an OpenShift Route.

* **OpenShift Native Tools**: Leverage OpenShift-native resources, such as
  `BuildConfig` and `ImageStream`, to build and manage the modified container
  image.

* **Image Layering**: Base the new container image on the currently deployed
  image, adding a new layer with the required modifications.

* **Custom Image Deployment**: Use the `OpenStackVersion` interface to apply
  the custom container image to the OpenStack deployment.

The Horizon image is used as an example in this document to demonstrate the
hotfix workflow. However, this process is applicable to any OpenStack service.

### Build the image

First, identify the container image currently in use. This image will serve as
the base layer for your patched version. The image reference retrieved here
will be used in the `FROM` statement of the `Dockerfile` within the `BuildConfig`.

```bash
$ oc -n openstack get Horizon -o json | jq '.items[0].spec.containerImage'
"quay.io/podified-antelope-centos9/openstack-horizon@sha256:17047ee14d355dfe02d86f09b52d254156b8a92b5cf8cd7c8d7b4b1579f3113b"
```

Create a manifest file named `horizon-hotfix.yaml` with the following content.
This defines an `ImageStream` to hold the patched image and a `BuildConfig` to
build it using OpenShift's internal build tools.

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: horizon-hotfix
  namespace: openstack
spec:
  lookupPolicy:
    local: true
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: horizon-hotfix
  namespace: openstack
spec:
  output:
    to:
      kind: ImageStreamTag
      name: horizon-hotfix:latest
  source:
    type: Docker
    dockerfile: |
      FROM quay.io/podified-antelope-centos9/openstack-horizon@sha256:17047ee14d355dfe02d86f09b52d254156b8a92b5cf8cd7c8d7b4b1579f3113b
      RUN dnf update -y
      RUN dnf install -y https://cbs.centos.org/kojifiles/packages/python-django-pyscss/2.0.3/1.el9s/noarch/python3-django-pyscss-2.0.3-1.el9s.noarch.rpm
    binary:
      asFile: ""
  strategy:
    type: Docker
    dockerStrategy:
      env:
        - name: NODE_ENV
          value: production
  triggers:
    - type: ConfigChange
```
Apply the manifest using the openshift client tool:

```bash
$ oc apply horizon-hotfix.yaml
```

Run the following command to confirm that the hotfix image has been
successfully pushed to the OpenShift internal registry:

```bash
$ oc get imagestream -n openstack

NAME             IMAGE REPOSITORY                                                                   TAGS     UPDATED
horizon-hotfix   default-route-openshift-image-registry.apps-crc.testing/openstack/horizon-hotfix   latest   26 seconds ago
```

To apply the new image, update the `OpenStackVersion` custom resource and
include the hotfix image under `spec.customContainerImage`. For example:

```yaml
spec:
  customContainerImage:
    horizonImage: image-registry.openshift-image-registry.svc:5000/openstack/horizon-hotfix:latest
```
For more information about the `OpenStackVersion` approach see [OpenStackVersion custom-images
document](version_updates.md#custom-images-for-other-openstack-services).

**Note:**
Replace the image path with your actual internal registry route if different.


### Expose the openshift-image-registry

By default, the OpenShift internal image registry (`openshift-image-registry`) is
not externally accessible. However, exposing it can be useful when non-cluster
components—such as `EDPM` nodes need to pull patched container images directly
(e.g., hotfixes built with OpenShift `BuildConfig` described in the previous
section).

This section explains how to expose the internal registry using a secure
`Route`, verify access, and pull images using `podman`.

1. Ensure the `openshift-image-registry` project exists and that the registry pods
and services are running:

```bash
$ oc get clusteroperator image-registry
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
image-registry   4.18.4    True        False         False      8d

$ oc get pods -n openshift-image-registry
NAME                                               READY   STATUS      RESTARTS   AGE
cluster-image-registry-operator-7769bd8d7d-4xwcp   1/1     Running     2          13h
image-registry-56f7846646-55tj4                    1/1     Running     0          11h
node-ca-98n8x                                      1/1     Running     1          12h
node-ca-jr5qn                                      1/1     Running     1          12h
node-ca-x24kt                                      1/1     Running     1          12h

$ oc get svc -n openshift-image-registry
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
image-registry            ClusterIP   172.30.80.87   <none>        5000/TCP    12h
image-registry-operator   ClusterIP   None           <none>        60000/TCP   13h
```

2. To make the registry reachable from outside the OpenShift cluster (e.g., from
   `EDPM` nodes using `podman`), enable the default `Route`:

```bash
 oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

3. Confirm the `openshift-image-registry` is reachable through a public `Route`:

```bash
$ oc get route -n openshift-image-registry
NAME            HOST/PORT                                                       PATH   SERVICES         PORT    TERMINATION   WILDCARD
default-route   default-route-openshift-image-registry.apps.ocp.openstack.lab          image-registry   <all>   reencrypt     None
```

**Note:**
The default route uses reencrypt TLS termination. This means the registry
presents a certificate that may not be trusted by external systems (like
podman) without additional setup.
For more information about the `openshift-image-registry` management, follow
the official [openshift
guide](https://docs.openshift.com/container-platform/4.16/registry/securing-exposing-registry.html)

4. Use the OpenShift credentials to authenticate with the exposed registry from
   an external host (e.g., EDPM node):

```bash
$ podman login -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps.ocp.openstack.lab
```

5. Verify you can pull the image that has been pushed via `BuildConfig`

```bash
$ podman pull default-route-openshift-image-registry.apps-crc.testing/openstack/horizon-hotfix
```

**Note:**

Append `--tls-verify=false` to the `podman` commands in case of issues with TLS cert verification
