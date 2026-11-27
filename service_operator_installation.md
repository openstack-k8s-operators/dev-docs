# How Service Operators Are Vendored and Installed

## Overview

openstack-operator is a "meta" operator: it installs and manages all of the
individual service operators (keystone-operator, heat-operator, cinder-operator,
etc.) that make up a podified OpenStack deployment. Unlike a typical OLM-based
setup, there are **no per-service `Subscription` or `CatalogSource` resources**
in the cluster for keystone-operator, heat-operator, and so on. OLM only
installs `openstack-operator` itself.

Instead, openstack-operator vendors each service operator's CRDs, RBAC, and
manager image reference at **build time**, bakes them into its own image, and
then applies them directly to the cluster at **runtime** using its own
controller logic (no OLM involved for the service operators). The only OLM
objects related to service operators that ever exist in the cluster are
legacy leftovers from older installs, which openstack-operator actively
cleans up.

This document walks through both halves of that process:

- [Build time](#build-time): how a service operator's bundle gets turned into
  files that live in the openstack-operator repo and image.
- [Runtime](#runtime): how the `OpenStack` CR controller uses those vendored
  files to create the service operator CRDs and Deployments in the cluster.

See also:
- [Building and Deploying a Custom OpenStack Operator](custom_openstack_operator.md)
  for the practical steps to test a service operator change end-to-end.
- [Building Operator Updates for OLM](olm_upgrades.md) for how openstack-operator
  itself is upgraded via OLM.

## Build time

At build time, openstack-operator pulls down every service operator's OLM
**bundle** image and extracts the pieces it needs. The set of bundles to pull,
and which version of each, is driven entirely by openstack-operator's `go.mod`.

### 1. The service operator list comes from `go.mod`

`hack/pin-bundle-images.sh` looks at the `openstack-k8s-operators/*-operator/api`
module dependencies in `go.mod` (excluding `openstack-operator` and
`lib-common`) and resolves each one to a `<name>-operator-bundle` image tag/SHA
in quay.io. This means the version of heat-operator that gets vendored is
whatever commit/tag is referenced by the `heat-operator/api` line in
openstack-operator's `go.mod` (or a `replace` directive pointing at a fork, see
[custom_openstack_operator.md](custom_openstack_operator.md)).

### 2. `make bindata` pulls and extracts each bundle

Running `make bindata` invokes `hack/sync-bindata.sh`, which for each bundle
returned by `pin-bundle-images.sh`:

1. `skopeo copy docker://<bundle-image> dir:...` pulls the bundle image.
2. Untars the bundle's manifest layer.
3. Extracts pieces of the bundle into the openstack-operator tree:
   - **CRDs** (any manifest containing `CustomResourceDefinition`) →
     `bindata/crds/`
   - **RBAC** (Roles, RoleBindings, ClusterRoles, ClusterRoleBindings, ServiceAccounts)
     parsed out of the bundle's CSV → `config/operator/bundle-rbac/heat-operator-*.yaml`
   - **Manager container image** parsed out of the CSV's deployment spec →
     written as a `RELATED_IMAGE_HEAT_OPERATOR_MANAGER_IMAGE_URL` env var in
     `config/operator/manager_operator_images.yaml` (and mirrored in
     `hack/export_operator_related_images.sh` for local/host runs via
     `make run-operator`)
   - **Webhook definitions** (only for infra-operator and
     openstack-baremetal-operator today) → `bindata/operator/<op>-webhooks.yaml`
   - **Services/Issuers/Certificates** needed by cert-manager →
     `bindata/services/<op>-services.yaml`

`make bindata` also rebuilds `bindata/crds/crds.yaml` from `config/crd` and
`bindata/operator/operator.yaml` (openstack-operator's own deployment
template) from `config/default`, and copies `config/operator/managers.yaml`
(the Go template used to render every *service* operator's Deployment) into
`bindata/operator/`.

All of the above is checked in — `bindata/`, `config/operator/bundle-rbac/`,
and `config/operator/manager_operator_images.yaml` are committed to the repo,
so a `go.mod` change plus `make bindata` is what actually pulls in a new
service operator version.

### 3. The vendored files are baked into the openstack-operator image

The `Dockerfile` copies the `bindata/` directory into the final image at
`/bindata` (`COPY --from=builder ${DEST_ROOT}/bindata /bindata`). At runtime
the controller reads from `BASE_BINDATA` (default `/bindata`), so whatever was
vendored at build time is what a given openstack-operator image will deploy.

### 4. `manager_operator_images.yaml` env vars land in the CSV

`make bundle` folds `config/operator/manager_operator_images.yaml` into the
openstack-operator CSV as `RELATED_IMAGE_*_OPERATOR_MANAGER_IMAGE_URL`
environment variables on the `openstack-operator-controller-manager`
container spec. This is the hand-off point to runtime: OLM (or a manual
Deployment edit) is what actually sets these env vars on the running pod.

## Runtime

Once openstack-operator (with its vendored `/bindata` and
`RELATED_IMAGE_*_OPERATOR_MANAGER_IMAGE_URL` env vars) is running in the
cluster, the `OpenStack` CR controller does the rest. All of the logic below
lives in `internal/controller/operator/openstack_controller.go`.

### 1. `SetupEnv()` reads the operator image env vars

On startup, `SetupEnv()` scans the process environment and builds a map of
service operator name → image:

- Any `RELATED_IMAGE_<NAME>_OPERATOR_MANAGER_IMAGE_URL` becomes
  `envRelatedOperatorImages["<name>"]` (e.g. `heat` →
  `quay.io/.../heat-operator@sha256:...`). This is how heat-operator's
  container image is selected.
- `OPERATOR_IMAGE_URL` sets openstack-operator's own image.
- Other `RELATED_IMAGE_*` vars (OpenStack service container images, e.g.
  glance-api) are collected separately and passed down as env vars to the
  relevant service operators.

`api/operator/v1beta1/openstack_types.go` defines the canonical list of
supported service operators (`OperatorList`, with names like
`HeatOperatorName = "heat"`). An operator only gets deployed if it both
appears in `OperatorList` and has a matching `RELATED_IMAGE_*` env var set.

### 2. `Reconcile()` renders and applies the vendored manifests

When the singleton `OpenStack` CR reconciles, `applyManifests()` does three
things:

- `applyCRDs()` renders everything under `bindata/crds` and applies it.
- `applyServices()` renders `bindata/services` (cert-manager Issuers,
  Certificates, metrics Services).
- `applyOperator()` renders `bindata/operator` — this is what creates the
  actual `<name>-operator-controller-manager` Deployments.

CRDs and Services are only re-applied when the openstack-operator container
image itself changes (`containerImageMatch`), to avoid re-rendering on every
reconcile. `applyOperator()`, however, runs on every reconcile.

### 3. `applyOperator()` builds a Deployment per service operator

For each entry in `OperatorList` that has a resolved image,
`applyOperator()` builds an `operator.Operator` struct (name, namespace,
replicas, image, env vars, resource limits/requests, tolerations), applying:

- Sensible defaults (replicas, CPU/memory limits).
- Per-operator env var tweaks (e.g. `ENABLE_WEBHOOKS`, `METRICS_CERTS`).
- Any user-supplied `OperatorSpec` overrides from
  `OpenStack.spec.operatorOverrides` (see `internal/operator/override.go`,
  `SetOverrides`). Note that overrides only cover replicas, resources, and
  tolerations — **there is no field to override an operator's image via the
  CR**. Image selection is exclusively controlled by the
  `RELATED_IMAGE_*_OPERATOR_MANAGER_IMAGE_URL` env vars on
  openstack-operator's own Deployment.

These `Operator` structs are fed into the `bindata/operator/managers.yaml`
Go template (`data.Data["ServiceOperators"]`), which is rendered and applied
via `renderAndApply()` → `bindata.RenderDir()` / `bindata.ApplyObject()`
(`internal/operator/bindata/render.go`, `apply.go`). The result is a
Deployment named `<name>-operator-controller-manager` with
`image: <the resolved RELATED_IMAGE_..._URL value>`.

Because this happens every reconcile, directly editing a service operator's
Deployment (e.g. `heat-operator-controller-manager`) in place will be
reverted on the next reconcile. To change which image is deployed, the
`RELATED_IMAGE_HEAT_OPERATOR_MANAGER_IMAGE_URL` env var on
`openstack-operator-controller-manager` must be changed instead (see
[custom_openstack_operator.md](custom_openstack_operator.md)).

### 4. Legacy OLM cleanup

`cleanupObsoleteResources()` and `postCleanupObsoleteResources()` exist
purely to migrate clusters that previously installed service operators via
OLM (individual `Subscription`/`CatalogSource`/CSV per operator). They delete
old `ClusterServiceVersion`, `Subscription`, and `InstallPlan` objects
belonging to service operators, and then the associated `operators.coreos.com`
`Operator` objects, once it's safe to do so. This is why the controller has
RBAC on `clusterserviceversions`, `subscriptions`, and `installplans` (`get`,
`list`, `delete` only) even though it never creates them for service
operators.

## Summary diagram

```
BUILD TIME (in openstack-operator repo/CI)
  go.mod (heat-operator/api version)
      |
      v
  hack/pin-bundle-images.sh  --> resolves heat-operator-bundle:<sha>
      |
      v
  make bindata -> hack/sync-bindata.sh
      |  skopeo copy + extract bundle
      |
      +--> bindata/crds/heat.openstack.org_*.yaml
      +--> config/operator/bundle-rbac/heat-operator-*.yaml
      +--> config/operator/manager_operator_images.yaml
      |      (RELATED_IMAGE_HEAT_OPERATOR_MANAGER_IMAGE_URL=quay.io/.../heat-operator@sha256:...)
      |
      v
  Dockerfile: COPY bindata /bindata
  make bundle: env vars folded into openstack-operator CSV
      |
      v
  openstack-operator image + bundle + catalog (pushed to registry)

RUNTIME (in the cluster)
  OLM installs openstack-operator (CSV sets RELATED_IMAGE_* env vars)
      |
      v
  openstack-operator-controller-manager pod starts -> SetupEnv()
      |  envRelatedOperatorImages["heat"] = <image>
      |
      v
  OpenStack CR reconciles -> applyManifests()
      +--> applyCRDs()      (bindata/crds, only on operator image change)
      +--> applyServices()  (bindata/services, only on operator image change)
      +--> applyOperator()  (every reconcile)
              |
              v
        renders bindata/operator/managers.yaml
              |
              v
        creates/updates heat-operator-controller-manager Deployment
        with image = envRelatedOperatorImages["heat"]
```
