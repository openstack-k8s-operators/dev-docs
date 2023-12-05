## Operator Lifecycle Manager (OLM)
General information and OLM intro can be found e.g.:

* https://docs.openshift.com/container-platform/4.12/operators/understanding/olm/olm-understanding-olm.html
* https://www.youtube.com/watch?v=aA6aEOfSG9g&t=2367s
* https://www.youtube.com/watch?v=5PorcMTYZTo

To deploy the operator using OLM three images are required, the operator image, a bundle image and the index or catalog. You can create the images manually or automated using the github actions in the operator repos.

## Creating images manually

```bash
export IMG="quay.io/<base>/keystone-operator:v0.0.1"
export BUNDLE_IMG="quay.io/<base>/keystone-operator-bundle:v0.0.1"
export CATALOG_IMG="quay.io/<base>/keystone-operator-index:v0.0.1"

make docker-build docker-push
make bundle bundle-build bundle-push
make catalog-build catalog-push
```

Also check the operator-sdk quickstart at https://sdk.operatorframework.io/docs/building-operators/golang/quickstart/

### OpenStack operator note for RHEL 8 non-root users

_(As noted in the section title, this only seems to apply to non-root users on RHEL 8)_

Building the OpenStack operator images locally requires `skopeo`, which itself must log into quay.io during its execution.  
Thus this requires authentication.  By default, `skopeo` looks for login credentials in a location that may or may not currently 
exist on your machine, and may or may not be accessible by non-root users even if it is present.  If `skopeo` can't access the
location with your current user, the build process will fail.  To prevent this snag, it is recommended that you...
```bash
export REGISTRY_AUTH_FILE=<path to auth.json>
```
...to point `skopeo` to JSON login credentials for quay.io.  A typical location for this file is `/home/<your user>/.docker/config.json`.

## Creating images using github actions

As part of the operators there is a [github action](https://github.com/openstack-k8s-operators/keystone-operator/blob/main/.github/workflows/build-keystone-operator.yaml) which builds and pushes the images to quay.io/openstack-k8s-operators.

This action uses the following organization action secrets to be able to build and push the images

| Secret | Description |
| --- | --- |
| `IMAGENAMESPACE` | namespace/organization name (e.g. openstack-k8s-operators or quay account) |
| `QUAY_USERNAME` | user name to use to login to quay |
| `QUAY_PASSWORD` | password for the quay account |
| `REDHATIO_USERNAME` | user name to use to login to the RH registry |
| `REDHATIO_PASSWORD` | password for the RH registry account |

This github action can also be used when working on your fork to build and push images to your private quay. To do so the following is required:

**In your quay.io**

* Recommended to create a robot account in quay.io under `Account Settings` -> `Robot Account` which then gets write access to the operator repositories. Using thing prevents from adding the main user account details in the github secrets. E.g. when name the robot account `ospk8s` the full user name will be `<username>+ospk8s`. The password can be seen in the `Options` -> `View Credentials` when the robot account got created.
* For each operator create the three repositories `<service-name>-operator`, `<service-name>-operator-bundle` and `<service-name>-operator-index`
* Ensure the robot account has write permissions to all three repositories

**In your github.com**

* create a fork of the operator repo
* enable github actions to run the forked repo
* in the forked operator repo settings create the following repo secrets in `Settings` -> `Secrets and Variables` -> `Actions`

| Secret | Description |
| --- | --- |
| `IMAGENAMESPACE` | your quay namespace |
| `QUAY_USERNAME` | the robot account user name to use |
| `QUAY_PASSWORD` | the robot account password for the use |
| `REDHATIO_USERNAME` | user name to use to login to the RH registry |
| `REDHATIO_PASSWORD` | password for the RH registry account |

When a branch gets now pushed to the forked repo, the github actions get triggered and all three images get build. The images get pushed with the commit ID as tag. Also a tag with `<branch name>-latest` points to the latest version. When deploying the operator for testing, it is recommended to use the commit ID tag as this is uniq and prevents issues when deploy multiple times tifferent versions.
