Writing KUTTL test for an operator
==================================

Kuttl is a tool for testing Kubernetes operators. Each kuttl test suite
consists of a series of test steps that might modify the state of the cluster
or check its state. A kuttl test step will use all yaml files that begin with
its numeric index. For example, if we had a file 00-deploy.yaml and
a 00-assert.yaml, kuttl would apply the changes described by the file
00-deploy.yaml and then wait until the state of the cluster matches the one
found in 00-assert.yaml. If it's found, then the step is successful. If after
the timeout, the state of the cluster does not match the assert, then the test
fails. Kuttl also supports errors files, which will check that the cluster does
not match the state described in the file (i.e tha inverse of an assert file).

What should be tested for in a kuttl suite depends on the details of the
operator itself. A good place to find more information is the controller.go
file for the operator, there you should find
hints on what resources does the operator create and/or manage. Another source
of information s the operator API where it list which attributes the operator
exposes. As an example, in keystone, one can find [here](https://github.com/openstack-k8s-operators/keystone-operator/blob/main/api/bases/keystone.openstack.org_keystoneapis.yaml) the properties of the API. Another good option would be to contact the people
listed as approvers in the operator's `OWNERS` file. Below are some example of
things to assert while writing the tests (they might not apply to all operators):

* Creation (the operator should deploy properly the service in question)
* Scale up and down (the operator should control properly the number of
replicas in the deployment and create and destroy pods as needed)
* Destroy (the operator should delete properly the resources deployed)
* Update (the operator applies some change to the resources deployed). As
an example, the keystone-operator can change the configuration of the
keystone service, which entails deleting the keystone pod and creating
a new one with the modified configuration.

Below are some conventions that are assumed for kuttl tests in
openstack-k8s-operators:

* There should be a kuttl-test.yaml file at the root of the operator
repo which contains the kuttl configuration.
* There should be `${OPERATOR_NAME}_kuttl` and `${OPERATOR_NAME}_kuttl_run` targets in `install_yamls`. The first will install all dependencies, run the  tests and cleanup, while the latter will simply run the tests, without further modifying the environment.
* There should be an assert file under
`operator-path/tests/kuttl/common/assert_sample_deployment.yaml`.
This assert file should be used to validate the deployment
of the operator. In parallel, a target called `${OPERATOR_NAME}_deploy_validate`
should be added to install_yamls. This new target should use a kuttl assert
command to check the validity of the operator deployment
against the `assert_sample_deployment.yaml` assert.

Below are some things to keep in mind while writing kuttl tests:

* Scripting can be used in assert files (useful if for example want to check some
parameter using regex, since that is not supported by kuttl). As
mentioned in the [kuttl docs](https://kuttl.dev/docs/testing/steps.html#shell-scripts),
the scripts used in an assert are passed to `sh -c`  to be
run, so the result can be environment dependent. For example, `sh` is often
symlinked to `bash`, but in many environments this will not be true, so be
careful to not use bash (or other shell) specific features.

* If you execute some `oc` command in a script, you'll need to pass `-n openstack` 
since the namespace is not gathered from the kuttl configuration for scripts
(the namespace is gathered for the yaml section of the assert if the `namespaced` option
is set to `True` in the kuttl configuration).

* Individual assert files can be applied using the `kubectl-kuttl assert` command.
This is useful to apply manually some changes to the cluster and debug the
assert file without needing to run the whole test suite. Example usage:
```
kubectl-kuttl assert --namespace openstack /path/to/02-assert.yaml --timeout 10.
```
Note that the assert command does not support having TestAssert blocks in the assert files (https://github.com/kudobuilder/kuttl/issues/221).

* There is also an `errors` command to debug errors files.


Tips and common mistakes
========================

* In a test step file you can create, update, delete multiple resources just
separate them with `---`.

* If you update an existing resource in a test step file you only need to
provide the fields that you want to change, kuttl will merge patch the existing
resource.

* You can use the TestStep in a test step file to execute a script to do
complex changes in the cluster. However TestSteps and other resource
definitions within the same test step file is applied in an random order by
kuttl. If you need a well defined sequence of steps then you need to use
separate step files with increasing indexes.

* In an assert file you can match multiple resources, just separate them with
`---`.

* The expected resource state in an assert file does not have to list all the
fields of the resource type. The fields that are not listed in the assert file
will be ignored during matching with the real resource. This way you can assert
only the fields relevant to the given test case.

* You can use the TestAssert resource in an assert or error file to run
shell scripts to assert complex things. However only a single TestAssert can
be used per file. If you add multiple TestAsserts then only the last one will
be executed and the rest is ignored by kuttl.

* If you use scripts in TestStep or TestAssert then add `set -euxo pipefail`
to the start of the script to ensure that no error is ignored.
