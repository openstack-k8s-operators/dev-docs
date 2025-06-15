# Testing local changes to lib-common

When developing changes in lib-common, prior to submitting a PR it's handy
to be able to test those changes with another operator that uses lib-common.
This topic describes how [run the operator locally](running_local_operator.md),
with it configured to use a local lib-common repository. This includes steps
for creating a full image of your operator with the lib-common changes built
in.

Note that this topic does not cover unit tests in lib-common.

## Prerequisites

* You forked the [lib-common](https://github.com/openstack-k8s-operators/lib-common) repository, and have a local clone of your fork
* You have an operator running locally that you intend to use to test your lib-common changes

## Procedure

1. Modify your operator's go.mod file(s)

Locate and edit your operator's go.mod files (when in doubt, try: `find
. -name go.mod`). Note the `require` entries for lib-common, and identify the
one associated lib-common code you're working on. This example will be for
changes to lib-common's **common** module.

```golang
require (
	...
	github.com/openstack-k8s-operators/lib-common/modules/common v0.1.1-0.20231001084618-12369665b166
	...
)
```

At the bottom of each go.mod file, add a `replace` line that references your
local lib-common repository. The reference may use an absolute path:

```
replace github.com/openstack-k8s-operators/lib-common/modules/common => /path/to/your/lib-common/modules/common
```

Or it may use a path relative to the go.mod file:

```
replace github.com/openstack-k8s-operators/lib-common/modules/common => ../lib-common/modules/common
```

2. Use tidy to refresh the corresponding go.sum file(s)

```shell
make tidy
```

3. Build and run your operator locally

```shell
make run-with-webhook
```

That's it! There's no need to `make build` in lib-common itself. Your local
lib-common code will be built into the locally built operator.

## Building an operator image with lib-common changes

In order to [build an operator](image_build.md) image that contains your
lib-common changes, those changes must first be pushed upstream to your
github fork.

1. Upload your changes to your lib-common fork github

* Commit your changes to your local lib-common repo
* Push the changes to your lib-common fork on github

You do not need to create a PR. You can do that later, once you're satisfied
with the changes.

2. Modify your operator's go.mod file(s) to reference the commit hash in your github repo

```
replace github.com/openstack-k8s-operators/lib-common/modules/common => github.com/YOUR_GITHUB/lib-common/modules/common commit_hash

```

3. Use tidy to refresh the corresponding go.mod and go.sum file(s)

```shell
make tidy
```

In addition to updating the go.sum file, tidy will update the go.mod file to
replace the commit hash with a "replacement version".

4. Continue the procedure for [building the operator image](image_build.md)
