# Bumping openstack-k8s-operator dependencies

The dependencies for all openstack-k8s-operators currently use
[psuedo-versions](https://go.dev/ref/mod#pseudo-versions). This includes the
dependencies for lib-common and all operator* repositories under the
openstack-k8s-operators github org.

## bumping all dependencies in a local operator repo

Each operator has a Makefile target called 'make force-bump'. Running this
target will bump the dependencies for all openstack-k8s-operator dependencies
in one step. After running this step if there are struct changes you may need
to re-run 'make manifests generate' to apply them. If you are using the
openstack-operator then you would also need to run a 'make bindata' step last
to update all the cached bindata files for the OpenStack initialization resource
(see more on this below).

If you prefer you can also manually update a single dependency using a command
like:
```
go get github.com/openstack-k8s-operators/glance-operator/api@main
```

## github actions to automatically bump dependencies

There are two github action workflows that can be used to automate bumping
dependencies for each operator repository:

- "Manually Trigger a Force Bump PR" action workflow can be executed on demand
to create or update a pull request that bumps operator dependencies for the
selected branch. NOTE: this action only runs if the workflow exists on that
branch (main and FR3 going forward)

- "Scheduled Force Bump PR" action workflow is executed automatically on
weekends. The actions run across the repositories in stages, a couple of pull
requests are generated each hour (so as not to overburden CI all at once). This
workflow runs on both the 'main' and last 'feature release' branches.

If there are failures or issues with the above you can manage and inspect the
github workflow executions in the 'actions' tab of each operator repo.

## special things about openstack-operator

OpenStack operator is our meta-operator and is responsible for installing all
other operators for the product via an initialization resource. In addition to
normal operator dependency bumps described above the following conventions and
considerations should be made:

- always run 'make bindata' after making any dependency changes to
openstack-operator or promoting the dependencies for any service operator within
openstack-operator. This needs to be in sync as openstack-operator is responsible
for creating the CRDs for all other service operators in the product. The bindata
files get built into the openstack-operator container directly (not the bundle).
This pattern is different than normal operator-sdk projects in that CRDs often
got installed by OLM directly there and this is no longer the case.

- we are using the go.mod file psuedoversion for each operator as the tag/pin for
each operator's controller-manager container. This is a convention that is used
upstream only on both 'main' and the 'feature release' branches. This means when
you bump the dependency of a service operator (keystone-operator, or
glance-operator, etc) in the openstack-operator project you are also bumping the
dependency of the service operator controller-manager container artifact. This works
because we tag all container images with the psuedoversion and thus can always find
the correct container image. When 'make bindata' executes it syncronizes all the
RELATED\_IMAGE variables to point to the correct container images accordinly using
the appropriate [sha/digest](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/operator/manager_operator_images.yaml)
so that offline airgapped testing is also fully supported upstream.

# Backwards incompatible shared golang function changes

NOTE: this applies to golang functions only not to API structs. There should
be no backwards incompatible golang struct changes that would break
existing deployments.

If you have backward incompatible function changes in any of the operators `/api`
modules, (e.g. changing the signature of a function in `/api` called by other
operators) then you need to land this changes in the dependency order of our
operators. See [OperatorDependencyBumpOrder](https://github.com/gibizer/openstack-k8s-status/blob/main/OperatorDependencyBumpOrder.md).

If you only change the main module of the operators then you can land them
in any order, but you should not forget that you still need to bump the version
in openstack-operator main `go.mod` file.

## Moving to a newer controller-runtime, k8s.io and golang versions

The controller-runtime and k8s.io dependencies needs to be in sync and they
should be close to the k8s version we use. E.g. controller-runtime 0.16
requires k8s.io 0.28 and it is matching with k8s 1.28 version.

Some of the controller-runtime minor bumps also require golang version bumps.
E.g. controller-runtime 0.16 requires golang 1.20, 0.17 requires 1.21.

Bumping the minor version of these dependencies often requires synchronized
effort across all of our operators, install_yamls and ci-framework. Below
I list the learnings we gathered during the bump from controller-runtime 0.14,
k8s.io 0.26 and golang 1.19 to controller-runtime 0.16, k8s.io 0.28, golang
1.20.

1. Before moving to a new golang version you need to ensure that
`ubi9/go-toolset` container available with the given golang version.

2. Due to our upstream build logic you need to land all the version bumps in
lib-common and the service operators first. Then you need to bump the the
dependencies in openstack-operator as a last step. The CI will be red until to
land the openstack-operator change. So agree on a day (or two) with the
community when such CI outage is acceptable.

3. Even if the kuttl and zuul based CI will be red on the service operator in
the meantime you can still run the envtest based functional tests to have some
level of confidence about the changes. If `/api` changes are involved you can
use `go.mod` replace lines pointing to unmerged dependencies to run functional
test before merge.

4. If there is backward incompatible changes in `/api` then you need to land
the bumps in operator dependency order. See above.

5. `go mod tidy` might not pick up all the changes in the `go.sum` files
automatically. You can simply delete all the `go.sum` files in the repo and
let `go mod tidy` or `make tidy` regenerate it. Sometimes you might need to
even to run `pre-commit run -a` and `make tidy` repeated to end up in a stable
`go.sum` state.
In some cases indirect dependencies moved to too new version due to too new
golang version was available in the environment. To properly clean such
indirect dependency version you can use the following commands to regenerate
all the indirect deps after you fixed the golang version:
```shell
rm go.sum  api/go.sum go.work go.work.sum
sed -i '/ indirect/d' go.mod api/go.mod
pre-commit run --all-files
```

6. If the golang version is changing then you have to make sure that
[install_yamls](https://github.com/openstack-k8s-operators/install_yamls/commit/e9cd3fcda0cf2205a25f56ca3af850150bb465a8) and
[ci-framework](https://github.com/openstack-k8s-operators/ci-framework/commit/5c138d4f734e600cde2e78859da21ab9fa835d33)
is using the newer golang version as well.

After the last openstack-operator change is landed and CI is back to green
there are some follow ups you can do. (Some can be done during the original
bump if you are feeling lucky):

1. Move the version pins in the
[global renovate config](https://github.com/openstack-k8s-operators/renovate-config/commit/f25b1bb774777111a4ecf4829532e5840176971a)
to the new maximums and [bump the golang version constraint](https://github.com/openstack-k8s-operators/renovate-config/commit/1b66fc3dd2466a13615a119a5433a81d9d34e022)
(probably ignored by renovate for now).

2. Update the
[`ENVTEST_K8S_VERSION` in the `Makefile`](https://github.com/openstack-k8s-operators/lib-common/commit/90181d385ef7972d3f5f2c36b20708888021d1bb)
of each repository so that it using the same version as the k8s version you
target now.

3. Bump the golangci-lint version in the ` .pre-commit-config.yaml` in each
repository to the newest version that supports the new golang version.
