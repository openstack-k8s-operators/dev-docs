# Deploying custom OpenStack service images

How to point a control-plane deploy at a **custom** OpenStack service image (for example one built from an unmerged OpenDev change). This guide covers CRC / local clusters, ci-framework jobs, and Zuul testprojects.

Building the image is out of scope here. For current development, prefer Source-to-Image (S2I) builds from [s2i-openstack-containers](https://github.com/openstack-k8s-operators/s2i-openstack-containers) (see that repository's [developer guide](https://github.com/openstack-k8s-operators/s2i-openstack-containers/blob/main/docs/developer-guide.md) and [image-mappings.yaml](https://github.com/openstack-k8s-operators/s2i-openstack-containers/blob/main/containers/image-mappings.yaml) for build targets and `OpenStackVersion` field names).

The older TCIB + RPM overlay examples that previously lived in this file are obsolete for current development, use S2I for service images.

Related in this repository:

- [OpenStackVersion](https://github.com/openstack-k8s-operators/dev-docs/blob/main/version_updates.md) — CR semantics, updates, and `customContainerImages`
- [Hotfix process](https://github.com/openstack-k8s-operators/dev-docs/blob/main/hotfix_process.md) — production-oriented hotfix packaging with the cluster registry



## Choose how to override images


| Goal                                                           | Approach                                                                                    |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| One (or a few) service images, keep the stock operator catalog | **OpenStackVersion** `spec.customContainerImages` (manual or via ci-framework)              |
| Full set of images from a content-provider / mirror registry   | ci-framework with `cifmw_update_containers_openstack: true` (not this guide's primary path) |
| Change a default URL inside the operator CSV                   | Rebuild openstack-operator / bundle / index (`RELATED_IMAGE_*`)                             |


Prefer OpenStackVersion (or the ci-framework variables that write it) over rebuilding the operator catalog.

## Manual OpenStackVersion (CRC / local cluster)

After the control plane exists, patch the OpenStackVersion CR in the OpenStack namespace (check with `oc get openstackversion -A`).

Fragment example (`spec` only):

```yaml
spec:
  customContainerImages:
    keystoneAPIImage: quay.io/<QUAY_NS>/openstack-keystone:<TAG>
    # openstackClientImage: quay.io/<QUAY_NS>/openstack-openstackclient:<TAG>
```

Use the field that matches your image (for example `glanceAPIImage`, `horizonImage`). Map names are documented under [Custom images for other OpenStack services](https://github.com/openstack-k8s-operators/dev-docs/blob/main/version_updates.md#custom-images-for-other-openstack-services). Cinder volume and Manila share backends use **maps** keyed by the resource name — see [Custom images for Cinder Volume](https://github.com/openstack-k8s-operators/dev-docs/blob/main/version_updates.md#custom-images-for-cinder-volume) and [Custom images for Manila Share](https://github.com/openstack-k8s-operators/dev-docs/blob/main/version_updates.md#custom-images-for-manila-share).

The Quay (or other) repository must be public, or the cluster must have pull credentials for that registry.

## ci-framework / testproject

Keep the published operator catalog whose `RELATED_IMAGE_*` defaults reference stock service images, and override only the service image(s) under test. Confirm what those defaults point at for your catalog before relying on a single-image override alone.

These variables come from ci-framework roles ([update_containers](https://github.com/openstack-k8s-operators/ci-framework/blob/main/roles/update_containers/README.md), [edpm_prepare](https://github.com/openstack-k8s-operators/ci-framework/tree/main/roles/edpm_prepare), [ci_gen_kustomize_values](https://github.com/openstack-k8s-operators/ci-framework/blob/main/roles/ci_gen_kustomize_values/README.md), [install_yamls](https://github.com/openstack-k8s-operators/ci-framework/blob/main/roles/install_yamls/README.md)).

Substitute your Quay namespace and image tag for `<QUAY_NS>` and `<TAG>`.

### Minimal variables for one custom service image

```yaml
vars:
  # Apply the generated OpenStackVersion CR. Role default is false; many
  # EDPM/component parent scenarios already set this to true.
  cifmw_update_containers: true

  # false = do not rewrite the full set of OpenStack service image URLs.
  # Custom images (and other explicitly configured images/maps) may still
  # be applied. true = also rewrite all services to
  # <registry>/<org>/openstack-*: <tag>.
  cifmw_update_containers_openstack: false

  cifmw_update_containers_custom_images:
    keystoneAPIImage: "quay.io/<QUAY_NS>/openstack-keystone:<TAG>"
    # optional second service, e.g. openstackclient overlay:
    # openstackClientImage: "quay.io/<QUAY_NS>/openstack-openstackclient:<TAG>"
```

If the parent job already uses a suitable published operator catalog and your custom image URL is fully qualified, the block above is enough.

### Catalog / registry overrides (when the parent defaults elsewhere)

If the parent defaults registry or catalog vars to a content-provider host, or you need a specific published operator index, add:

```yaml
vars:
  # When not running a content provider, override parent defaults that
  # point at "{{ content_provider_registry_ip }}:5001".
  cifmw_update_containers_registry: "quay.io"

  # Example: published upstream openstack-operator index (public on Quay).
  # Confirm that this index's RELATED_IMAGE_* values match the stock
  # images you expect before relying on a single-image override.
  # install_yamls accepts any cifmw_install_yamls_vars_patch* map.
  cifmw_ci_gen_kustomize_values_ooi_image: >-
    quay.io/openstack-k8s-operators/openstack-operator-index:latest
  cifmw_ci_gen_kustomize_values_sub_channel: alpha
  cifmw_install_yamls_vars_patch_s2i:
    OPENSTACK_IMG: >-
      quay.io/openstack-k8s-operators/openstack-operator-index:latest
    OPERATOR_CHANNEL: alpha
```

Do not combine this single-image override path with a content-provider workflow that sets `content_provider_os_registry_url`: `edpm_prepare` then forces `cifmw_update_containers_openstack: true` and rewrites all OpenStack service images from that registry.

### Backend maps (Cinder volume / Manila share)

`cinderVolumeImages` and `manilaShareImages` are **maps** keyed by the CinderVolume / ManilaShare **resource name** (the map key under `spec.cinder.template.cinderVolumes` / `spec.manila.template.manilaShares`, for example `pure-fc` or `share0`), not by the Cinder/Manila backend or driver name from service config (for example `volume_backend_name=pure`). Using a backend/driver name as the map key misses the lookup and falls back to the default image. Scalar entries in `cifmw_update_containers_custom_images` are not enough by themselves.

Even with `cifmw_update_containers_openstack: false`, some jobs still inject backend maps from registry defaults. If those backends matter, set or clear them explicitly via the prepare role variables (for example `cifmw_edpm_prepare_cindervolumes` / `cifmw_edpm_prepare_manilashares`, and any migration aliases documented in the `edpm_prepare` / `update_containers` READMEs). See also [version_updates.md](https://github.com/openstack-k8s-operators/dev-docs/blob/main/version_updates.md) for the CR field shapes.

## Verify

Resolve the OpenStackVersion resource name in your namespace first (`oc get openstackversion -A`). Examples below use `controlplane` in namespace `openstack`.

```bash
oc -n openstack get openstackversion controlplane \
  -o jsonpath='{.spec.customContainerImages.keystoneAPIImage}{"\n"}'
oc -n openstack get deploy,sts -o wide | grep -i keystone
# Requested image string from the pod spec:
oc -n openstack get pod -l service=keystone -o \
  jsonpath='{.items[*].spec.containers[*].image}{"\n"}'
# Runtime-resolved image (typically includes the digest):
oc -n openstack get pod -l service=keystone -o \
  jsonpath='{.items[*].status.containerStatuses[*].imageID}{"\n"}'
```

Expect your custom image reference on the OpenStackVersion field and in the pod spec. Use `status.containerStatuses[*].imageID` to confirm the runtime-resolved digest. Confirming the image reference does not prove the intended code change is present; exercise the patched behavior as well (or inspect the installed package/module inside the pod when that is meaningful for your change).

## Optional: publish via openshift-image-registry

If you prefer the cluster's internal registry instead of a public Quay repository (common on CRC / lab OpenShift), import a locally available image and reference the ImageStream URL from OpenStackVersion.

1. Confirm the registry is running:

```bash
oc get pods,svc -n openshift-image-registry
```

1. Expose a public route (if needed for `podman push` from the host):

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --type=merge --patch '{"spec":{"defaultRoute":true}}'
oc get route -n openshift-image-registry
```

See the OpenShift documentation on [exposing the registry](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/registry/securing-exposing-registry).

1. Log in, tag, and push (replace host and image names). `LOCAL_IMAGE` may be a Quay URL or a local tag such as `localhost/openstack/openstack-keystone:<TAG>`:

```bash
REGISTRY_HOST=$(oc get route default-route -n openshift-image-registry \
  -o jsonpath='{.spec.host}')
oc whoami -t | podman login --username "$(oc whoami)" --password-stdin "${REGISTRY_HOST}"
# Prefer trusting the registry CA in the host / podman trust store.
# Only on an isolated lab with an untrusted cert, you may append
# --tls-verify=false to login/push. That disables TLS verification and
# can expose credentials or image content to a spoofed registry.

LOCAL_IMAGE=quay.io/<QUAY_NS>/openstack-keystone:<TAG>
podman tag "${LOCAL_IMAGE}" \
  "${REGISTRY_HOST}/openstack/openstack-keystone:<TAG>"
podman push "${REGISTRY_HOST}/openstack/openstack-keystone:<TAG>"
```

1. Confirm the ImageStream and take the in-cluster pull URL:

```bash
oc -n openstack get is openstack-keystone
oc -n openstack get is openstack-keystone -o jsonpath='{.status.dockerImageRepository}{"\n"}'
# Optional digest form:
# oc -n openstack get is openstack-keystone -o json | \
#   jq -r '.status.tags[] | select(.tag=="<TAG>") | .items[0].dockerImageReference'
```

Typical in-cluster form: `image-registry.openshift-image-registry.svc:5000/openstack/openstack-keystone:<TAG>`.

1. (Optional) Remove the default route when finished:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --type=merge --patch '{"spec":{"defaultRoute":false}}'
```

Use that URL in `customContainerImages` as in the sections above. For a fuller CRC-oriented hotfix packaging flow, see [hotfix_process.md](https://github.com/openstack-k8s-operators/dev-docs/blob/main/hotfix_process.md).

## Optional: personal openstack-operator index

Only if you must change a default URL inside `RELATED_IMAGE_*` / the catalog:

1. Build and push the custom service image.
2. Point the matching entry in openstack-operator `config/operator/default_images.yaml` at your image URL.
3. Rebuild and push openstack-operator, bundle, and index.
4. Point the job or local deploy at your custom catalog image. If you still use ci-framework image updates, keep `cifmw_update_containers_openstack: false` (and set `cifmw_update_containers: true` when applying custom overrides).

You are still only changing a URL string in the catalog, not a container source pin.
