# Bumping dependencies

If you have backward incompatible changes in any of the operators `/api`
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

4. While test-operator is not part of the openstack-operator bundle it make
sense to do the same version bumps in the repo as well.