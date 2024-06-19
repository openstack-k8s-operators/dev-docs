# Release and Branching

## Outline

We release set of operators as a single product called the 'OpenStack Operator'. The openstack-operator itself
acts as a meta operator. It contains CRDs which can be used to create an OpenStackControlplane in a single action.
Additionally, installation of of a specific version of the openstack-operator drives the installation of all
openstack service operators via OLM (Operator Lifecycle Manager) dependencies. This document aims to summarize the
relationship between how these components are released and branched upstream.

## Artifacts

For each operator/repository upstream we build 3 artifact/containers:

* operator container
* bundle
* catalog/index image (upstream only)

The last artifact above (catalog/index images) are provided as a convience upstream as a means to install ad-hoc sets of
OpenStack services operators via projects like install_yamls. They are not used directly in the productized openstack-operator.
The operator container and bundle are used in all deployment cases whether you install individual operators, or as a set via
the meta openstack-operator.

These containers are tagged and uploaded to quay.io under the openstack-k8s-operators organization. Each container gets tagged with the following tags:

* branch name (or 'main')
* github SHA
* SHA256 digest of the container image itself

All of the operators under openstack-k8s-operators have the above artifacts and contains. The bundle for the openstack-operator itself is a special case described below.

## Bundles

Each operator builds its own bundle that contains manifests including CRDs and the CSV (Cluster Service Version).

All the service operators create a "normal" bundle in this fashion. The openstack-operator's bundle is a special case and does
the following extra steps:

* It contains a dependencies.yaml file which drives OLM dependencies to install all other OpenStack service operators
* When the bundle is generated it runs a script to *merge* all the ENV variables from each service operators controller-manager. These ENV variables contain important webhook defaults that are needed within the OpenStack operator.
* A csv-merger is used to merge all the ENV variables harvested from the above.

NOTE: We no longer merge all the CSV for all openstack service operators because we hit the bundle size limit. This is a limitation due to how OLM relies on ConfigMaps when unpacking bundles. (separate bundles are required for OpenStack operators at this time)

## How go.mod is used to pin the go dependencies and control the service operator containers that get installed

Each service operator that is integrated into the openstack-operator has an entry in the go.mod for the openstack-operator.  We required the use of [psuedoversions](https://go.dev/ref/mod#pseudo-versions) in the go.mod file for all openstack service operator dependencies. In addition to controlling how the golang dependency gets installed the psuedoversion controls which bundle is used and installed with the openstack-operator and acts as a means to control and promote new service operator builds to be used.

The logic driving this integration exists in the openstack-operators hack/pin-bundle-images.sh script.
This script is used both during bundle build and again when the catalog/index image to obtain the correct
container image from a go.mod entry that looks like this:
```golang
    github.com/openstack-k8s-operators/cinder-operator/api v0.3.1-0.20231114160640-3c5c40e6cc3a
```
The last part of the psuedoversion corresponds to a git commit sha and can be used to pull in
dependencies from the associated bundle for that service operator.
In this case: quay.io/openstack-k8s-operators/cinder-operator-bundle:3c5c40e6cc3aba6e287b94d26770519f76ae9528 would get used.

## Tagging submodules for api/apis (nested modules)

Each service operator has either an 'api' or an 'apis' directory that contains the API structs, webhooks, and some common helpers for that operator. These directories have a separate nested go.mod file in order to minimimize the dependencies involved in using
the CRD structs in external operators.

NOTE: there are tentative plans to combine things into a common 'api' project at a later date.

The api submodule directories can be periodically tagged (independent from the top level go.mod for an operator) to signify
a major release or development milestone. It should be noted that doing so will temporarily cause 'go mod' to default to
a tag when trying to obtain the latest version of an API. Example:
If you have recently tagged the 'api' directory on the main branch with a command like:
```bash
   git tag api/v0.2.0
```
And then you run this command:
```bash
    go get github.com/openstack-k8s-operators/cinder-operator/api@main
```
You could end up with this in your go.mod file(s):
```golang
    github.com/openstack-k8s-operators/cinder-operator/api v0.2.0
```

If this happens it is recommended that you make one extra commit to the api dir to flip it back to a psuedoversion.

NOTE: This is a behavior of 'go mod' that would be nice to force to use psuedoversions. TODO(look into how we might force this)

Our convention is currently to tag/bump service operator API submodules on 'main' only when an upstream release branch gets created.

# Tagging an operator

Tagging the operator codebase itself should not impact the go submodules and thus can be done at any time on main or from a release branch.

# Installing the latest openstack-operator from main or a branch

For the main branch you can always install the latest openstack-operator by using the following catalog:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opentack-operator-index
  namespace: openstack-operators
spec:
  image: quay.io/openstack-k8s-operators/openstack-operator-index:latest
  sourceType: grpc
```

For a release branch you can always install the latest openstack-operator by using the following example catalog (for dev-preview2):

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opentack-operator-index
  namespace: openstack-operators
spec:
  image: quay.io/openstack-k8s-operators/openstack-operator-index:dev-preview2-latest
  sourceType: grpc
```

For any given index the bundles that are installed are controlled by the references in the go.mod for the openstack-operator itself.
