## Building custom OpenStack images

This document provides an example of using TCIB to build
a modified version of the openstack-glance-api container.
It then provides an example of how to publish it to a personal
quay.io account and deploy it for testing in OCP.

### Build an image with TCIB

[TCIB](https://github.com/openstack-k8s-operators/tcib)
can be used to build the images deployed by the
openstack-k8s-operators. In this example we will use TCIB to
build an openstack-glance-api with
[patch 924824](https://review.opendev.org/c/openstack/glance/+/924824)
applied.

Follow the
[setup](https://github.com/openstack-k8s-operators/tcib/tree/main?tab=readme-ov-file#setup)
and then the
[Building images with local changes](https://github.com/openstack-k8s-operators/tcib/tree/main?tab=readme-ov-file#building-images-with-local-changes)
guides. In this example `containers.yaml` is modified to contain only
the glance image.

```
$ cat containers.yaml
container_images:
  - imagename: quay.io/podified-antelope-centos9/openstack-glance-api:current-podified
$
```
As per [glance-api.yaml](https://github.com/openstack-k8s-operators/tcib/blob/main/container-images/tcib/base/os/glance-api/glance-api.yaml#L12),
TCIB builds the container by running a series of commands including
the installation of the `glance-api` package with `dnf`. There are
many ways to add the patch but in this example we will append the
following command to the end of the list of commands.
```
curl -k -L https://raw.githubusercontent.com/openstack/glance/ee7e96f06af741bb34bedac18fa2c4616fcc3905/glance/location.py -o /usr/lib/python3.9/site-packages/glance/location.py
```
When the following command is run:
```
sudo openstack tcib container image build \
   --config-file containers.yaml \
   --repo-dir /tmp/repos/ \
   --tcib-extras tcib_package=
```
an Ansible playbook is run and a log is generated showing the image
build location. The log should show if the additional command to apply
the patch was run.
```
$ grep curl -B 1 -A 5 /tmp/container-builds/ac88bc87-c9d7-4270-98c7-8b0b5e64a6fc/base/os/glance-api/glance-api-build.log
STEP 7/10: RUN sed -i -r 's,^(Listen 80),#\1,' /etc/httpd/conf/httpd.conf &&  sed -i -r 's,^(Listen 443),#\1,' /etc/httpd/conf.d/ssl.conf
STEP 8/10: RUN curl -k -L https://raw.githubusercontent.com/openstack/glance/ee7e96f06af741bb34bedac18fa2c4616fcc3905/glance/location.py -o /usr/lib/python3.9/site-packages/glance/location.py
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 27092  100 27092    0     0   117k      0 --:--:-- --:--:-- --:--:--  117k
STEP 9/10: USER glance
STEP 10/10: LABEL "tcib_build_tag"="current-podified"
$
```
The same log will show where the new image was built.
```
$ grep COMMIT /tmp/container-builds/a7a02754-67e2-4c22-b479-196f8b7c30cb/base/os/glance-api/glance-api-build.log
COMMIT localhost/podified-master-centos9/openstack-glance-api:current-podified
$
```
Confirm the new image's local location.
```
$ sudo podman images | grep localhost
localhost/podified-master-centos9/openstack-glance-api   current-podified                           2639aa0e9581  16 hours ago  961 MB
localhost/podified-master-centos9/openstack-os           current-podified                           967591dc4b05  16 hours ago  349 MB
localhost/podified-master-centos9/openstack-base         current-podified                           85c7395479db  16 hours ago  209 MB
$
```
### Publish the image to a quay account

Tag the image.
```
sudo podman tag \
  localhost/podified-master-centos9/openstack-glance-api:current-podified \
  quay.io/$USERNAME/podified-master-centos9/openstack-glance-api:current-podified
```
Use the quay.io web interface to create an encrypted password and
confirm it works with `podman login`.
```
sudo podman login -u="$USERNAME" -p="************" quay.io
```
Push the image.
```
sudo podman push \
  quay.io/$USERNAME/podified-master-centos9/openstack-glance-api:current-podified
```
Verify you can see the image at a URL like 
https://quay.io/repository/$USERNAME/podified-master-centos9/openstack-glance-api?tab=tags&tag=current-podified

Use the `Settings` tab to make the image public so that the OCP can
download it.

Use the `fetch tag` option copy and paste an `podman pull` command
with the URL of the new image.

Follow the
[OpenStackVersion custom-images
document](version_updates.md#custom-images-for-other-openstack-services)
and use the URL to have OCP deploy the new image for testing.

Once the image is published use a command like
`oc rsh --container glance-api glance-3d293-default-external-api-0`
to get inside the new glance pod and then examine
`/usr/lib/python3.9/site-packages/glance/location.py`
to see if the patch was correctly applied.

### Import the image in openshift-image-registry

When the image is built using `tcib`, it is possible to use the `openshift-image-registry`
to push it and build a new `ImageStream` that can be consumed by the `OpenStackControlPlane`
through the `OpenStackVersion` approach described in the [OpenStackVersion custom-images
document](version_updates.md#custom-images-for-other-openstack-services).
To import the new local image in the `openshift-image-registry` run the following commands:

1. Verify the openshift-image-registry project exists and there is an associated service:

```bash
$ oc get pods -n openshift-image-registry
NAME                                               READY   STATUS      RESTARTS   AGE
cluster-image-registry-operator-7769bd8d7d-4xwcp   1/1     Running     2          13h
image-pruner-28719360-6pvb2                        0/1     Completed   0          11h
image-registry-56f7846646-55tj4                    1/1     Running     0          11h
node-ca-98n8x                                      1/1     Running     1          12h
node-ca-jr5qn                                      1/1     Running     1          12h
node-ca-x24kt                                      1/1     Running     1          12h

$ oc get svc -n openshift-image-registry
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
image-registry            ClusterIP   172.30.80.87   <none>        5000/TCP    12h
image-registry-operator   ClusterIP   None           <none>        60000/TCP   13h
```

2. To be able to push the local image using `podman`, expose the service through
a public `Route`:

```bash
 oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

3. Verify the `openshift-image-registry` is reachable through a public `Route`:

```bash
$ oc get route -n openshift-image-registry
NAME            HOST/PORT                                                       PATH   SERVICES         PORT    TERMINATION   WILDCARD
default-route   default-route-openshift-image-registry.apps.ocp.openstack.lab          image-registry   <all>   reencrypt     None
```

**Note:**

For more information about the `openshift-image-registry` management, follow the
official [openshift guide](https://docs.openshift.com/container-platform/4.16/registry/securing-exposing-registry.html)

4. Login to the exposed registry:

```bash
$ podman login -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps.ocp.openstack.lab
```

5. Tag and push the local image built using `tcib`:

```bash
$ podman tag quay.io/$USER/podified-master-centos9/openstack-glance-api:mytag image-registry.apps.ocp.openstack.lab/openstack/openstack-glance-api:mytag

$ podman push default-route-openshift-image-registry.apps.ocp.openstack.lab/openstack/openstack-glance-api:mytag
```

**Note:**

Append `--tls-verify=false` to the `podman` commands in case of issues with TLS cert verification

6. Verify the new `ImageStream` is present in the `openstack` namespace:

```bash
$ oc get is
NAME                   IMAGE REPOSITORY                                                                               TAGS              UPDATED
openstack-glance-api   default-route-openshift-image-registry.apps.ocp.openstack.lab/openstack/openstack-glance-api   mytag,latest   24 minutes ago
```

7. (Optional) Remove the `Route` from the `openshift-image-registry` as we import the image
using the internal `Service`

```bash
 oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":false}}' --type=merge
```

8. Get the image address in the form of `registry:<port>/<project>/<image>`:

```bash
$ oc get is openstack-glance-api -o json | jq '.status.dockerImageRepository'

"image-registry.openshift-image-registry.svc:5000/openstack/openstack-glance-api"
```

You can refer the image via sha by changing the above to match `dockerImageReference`, for example:

```bash
$ oc get is openstack-glance-api -o json | jq '.status.tags[1] | .items[0] | .dockerImageReference'
"image-registry.openshift-image-registry.svc:5000/openstack/openstack-glance-api@sha256:a36a9ae074dc3ea0674b88351589c0a84bd264d30ef62a0c42bab5081c0d3b4c"
```

Follow the
[OpenStackVersion custom-images
document](version_updates.md#custom-images-for-other-openstack-services)
and use the retrieved image URL (either image:tag or using the sha reference)
to have OCP deploy the new image for testing.

Once the image is published use a command like
`oc rsh --container glance-api glance-3d293-default-external-api-0`
to get inside the new glance pod and then examine
`/usr/lib/python3.9/site-packages/glance/location.py`
to see if the patch was correctly applied.
