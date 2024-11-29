# EnvTest

EnvTest is a set of tools allowing you to test your operator in isolation.

What is it good for:
* Testing a single operator in isolation without the need to mock out any
  external dependencies. You still need to simulate the *behavior* of your
  external interfaces but you can use the same interface definitions (CRDs)
  in the test that in production.
* Testing both happy and failure scenarios. You can simulate that a Job fails
  or that KeystoneAPI is not Ready. You can define order of external state
  changes, e.g. TransportURL becomes ready before MariaDBDatabase.
* Testing at every commit or even at every local changes you make as single
  test cases can be run in seconds and full suites can run in a minute.
* Reproducing failures for debugging. If you know the sequence of events that
  causes a failure you can write a test case for it and attach the debugger
  while the case runs to troubleshoot the issue.


What it is not good for:
* Testing integration between different operators as only your operator runs
  in the test environment.
* Testing the integration between an operator and k8s. You can assert if
  your operator created k8s resources properly but those resources will not be
  reconciled so you cannot assert k8s behavior.
* Testing internal implementation details of your operator. While you can
  access the Reconciler instance from the test code, you should only use the
  CRD interface of your operator.
  We have one exception. The lib-common helpers
  has no CRD interfaces but they also create k8s resources. So for example
  in the
  [test for the Job helpers](https://github.com/openstack-k8s-operators/lib-common/blob/1db7d191885e82ed25e51a34536c601a9082275f/modules/common/test/functional/job_test.go#L71-L85)
  we call the lib-common golang functions directly,
  but still asserting the resulted k8s Job via the k8s APIs.

### Why do not just use kuttl for all testing?

While kuttl seems to be a good choice for integration testing multiple
operators in a single environment to ensure that the happy path works, it has a
list of trade offs using it for testing a single operator:
* kuttl will be significantly slower than envtest as it needs every operator to
  start and reconcile. Also it needs to wait for all the external events to
  happen E.g. Jobs to finish, DBs to be created etc.
* kuttl will need a full k8s / OCP environment so that all the operators
  including OCP and k8s can run. This will require more CPU and Memory
  than EnvTest. For example, I can run 4 parallel EnvTest executions in a local
  laptop, but only a single all in one OCP fits into a laptop.
* kuttl works based on matching yaml files. There is no built in support for
  integer comparison or regexp match. You need to call out to bash for that
  which breaks up the test case to different files and different languages.
  EnvTest test cases are written in golang so it can use the whole golang
  ecosystem to implement test logic.
* with kuttl, it is hard or even impossible to simulate failure scenarios. E.g.
  simulating that KeystoneAPI is not Ready needs some way to "break" the
  configuration of the keystone deployment.
* debugging with kuttl is hard as you cannot put a breakpoint in the test code.

## Test env architecture
```
   ┌─────────────┐
   │suite_test.go│
   └┬────────────┘
    │       starts as processes              ┌────┐
    ├────────────────────────────────┬──────►│etcd│
    │                                │       └──┬─┘
runs│as goroutine                    ▼          │
    │                         ┌──────────────┐  │
    │                         │kube-apiserver├──┘
    │  ┌──────────────────┐   └──────────────┘
    ├─►│controller-manager│     ▲  ▲  ▲
    │  │ Reconcile()      ├─────┘  │  │
    │  └──────────────────┘        │  │
    │                              │  │
    │  ┌────────┐                  │  │
    ├─►│webhooks├──────────────────┘  │
    │  └────────┘                     │
    │                                 │
    │  ┌──────────────┐               │
    └─►│test scenarios├───────────────┘
       └──────────────┘
```

## Implementation

### Gomega
[Gomega](https://github.com/onsi/gomega) is a rich assert library to express
things like:
```golang
    novaAPI := GetNovaAPI(name)
	Expect(novaAPI.Status.ReadyCount).To(BeNumerically(">", 0))
```

Also it has a way to assert things asynchronously. Which is very useful for
our case where `Reconcile()` runs in an separate goroutine from the test
code
```golang
	Eventually(func(g Gomega) {
		novaAPI := GetNovaAPI(name)
		g.Expect(novaAPI.Status.ReadyCount).To(BeNumerically(">", 0))
    }, timeout, interval).Should(Succeed())
```

### Ginkgo

[Ginkgo](https://github.com/onsi/ginkgo) is a test framework supporting the BDD
style. It provides a way to organize our test cases in to suites and scenarios.

```go
var _ = Describe("NovaAPI controller", func() {
	When("NovAPI is created", func() {
		BeforeEach(func() {
			DeferCleanup(th.DeleteInstance, CreateNovaAPI(name, GetDefaultNovaAPISpec()))
		})

		It("creates a StatefulSet for the nova-api service", func() {
			ss := th.GetStatefulSet(novaNames.APIStatefulSetName)
			Expect(int(*ss.Spec.Replicas)).To(Equal(1))
		})

		It("creates KeystoneEndpoint", func() {
			keystoneEndpoint := th.GetKeystoneEndpoint(name)
			endpoints := keystoneEndpoint.Spec.Endpoints
			Expect(endpoints).To(
                HaveKeyWithValue("internal", "http://nova-internal.openstack.svc:8774/v2.1"))
		})
    })
```

#### Why BDD and not pure unit test style?

There are no technical reasons. You can use EnvTest (and Gomega) with the pure
golang unit test framework or [testify](https://github.com/stretchr/testify).
See a trial
[here](https://github.com/gibizer/nova-operator/compare/7d5e6ef068f18192e45830582e8d56ce4c185576...363289bd24738c320314717219e12d6550c42bf9).
The reason we ended up with Ginkgo and BDD is that all the examples in
kubebuilder and Operator SDK docs are using Ginkgo.

### Simulating k8s dependencies

In EnvTest your operator is running in isolation, so the test code needs to
simulate the other operators and OCP / k8s behaviors the operator under test
depends on. For example nova-operator needs a k8s Service to exists that
represents a running MariaDB service to create nova's DB instances. But there
is no mariadb-operator running in the test environment and nothing has created
the MariaDB CR, so no Service resource exists. Fortunately creating CR or a k8s
resource from the test code is simple:
```golang
	serviceName := "openstack"
	service := &corev1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      serviceName,
			Namespace: namespace,
			Labels: map[string]string{
				"app": "mariadb",
				"cr":  "mariadb-openstack",
			},
		},
		Spec: corev1.ServiceSpec{
			Ports: []corev1.ServicePort{{Port: 3306}},
		},
	}
	Expect(k8sClient.Create(ctx, service)).Should(Succeed())
```
Note that you can create any built in k8s resource or CRDs, but there won't be
any operator reconciling them. So the above `Service` will only exist in the
API but no real service will run behind it.

Similarly if your test creates a CR, implemented by another operator, and then
wait for the readiness of such CR then your test needs to make sure that such
readiness happens. As all these are expressed as Status fields on the CR the
test simply needs to set the Status fields of the CR. For example nova-operator
creates a k8s Job to run DB sync then waits for the Job to succeed. So the
test code needs to simulate such success:
```golang
	job := GetJob(name)
	job.Status.Succeeded = 1
	job.Status.Active = 0
	Expect(k8sClient.Status().Update(ctx, job)).To(Succeed())
```
Note that Status field can be set via the Status client.

There is one catch though. If you want to manipulate resources that are not
built into the kube-apiserver, e.g. CRDs defined by other operators, then
EnvTest needs to load the CRD definitions. See the
[suite_test.go](https://github.com/openstack-k8s-operators/nova-operator/blob/212dd0094c9bd2ef0dcfebb39eab2ac5881a4a37/test/functional/suite_test.go#L95-L125)
in nova-operator how to do that.

### Simulating OpenStack services

If you operator needs to call OpenStack service APIs during reconciliation then
you need to simulate those OpenStack services. For example the
keystone-operator needs to create users in keystone when reconciles a
KeystoneService CR. The keystone-operator implements that by calling the
OpenStack Keystone API via the gophercloud OpenStack SDK. So in the
keystone-operator test env we need to have some way to catch those API requests
and respond to them. The lib-common's test-operators module has a Keystone API
fixture to help simulating keystone's behavior.

Note that if your operator only depends on the KeystoneAPI, KeystoneService,
and KeystoneEndpoint CRDs to interact with the OpenStack Keystone API then you
don't need to simulate keystone itself, therefore you don't need this fixture.
You only need to simulate the success of these CRDs as described in the
previous section.

### Common helpers

There is a set of common test functionality that lives in various modules:
* [`lib-common/module/common/test/helpers`](https://github.com/openstack-k8s-operators/lib-common/tree/main/modules/common/test/helpers)
package in common module for non openstack-k8s-specific helpers:
  * Generic asserts functions like `ExpectCondition` that checks the status
  conditions of any openstack CRD
  * Helpers for managing resources like `CreateSecret` and `DeleteConfigMap`
  * Helpers for simulating external events like `SimulateJobSuccess`

  See [docs](https://pkg.go.dev/github.com/openstack-k8s-operators/lib-common/modules/common@main/test/helpers)
  for the full list.

* [`lib-common/module/test`](https://github.com/openstack-k8s-operators/lib-common/tree/main/modules/test)
module serves as a base for the openstack-k8s-specific helpers and fixtures and
contains generic CRD helpers as well as helper for the certmanager operator.
See [docs](https://pkg.go.dev/github.com/openstack-k8s-operators/lib-common/modules/test@main/)
  for the full list.

* [`keystone-operator/api/test/helpers`](https://github.com/openstack-k8s-operators/keystone-operator/tree/main/api/test/helpers)
contains helpers for the keystone CRDs.
See [docs](https://pkg.go.dev/github.com/openstack-k8s-operators/keystone-operator/api@main/test/helpers)
  for the full list.

* [`infra-operator/api/test/helpers`](https://github.com/openstack-k8s-operators/infra-operator/tree/main/apis/test/helpers)
contains helpers for the TransportURL and Memcached CRDs.
See [docs](https://pkg.go.dev/github.com/openstack-k8s-operators/infra-operator/apis@main/test/helpers)
  for the full list.

* [`mariadb-operator/api/test/helpers`](https://github.com/openstack-k8s-operators/mariadb-operator/tree/main/api/test/helpers)
contains helpers for the MariaDB and MariaDBDatabase CRDs.
See [docs](https://pkg.go.dev/github.com/openstack-k8s-operators/mariadb-operator/api@main/test/helpers)
  for the full list.

## Execution

1. Get the envtest helper
   ```make
   GOBIN=$(LOCALBIN) go install sigs.k8s.io/controller-runtime/tools/setup-envtest@latest
   ```

2. Ensure `kube-apiserver` and `etcd` binaries available, the envtest helper
from step 1 takes care of this

   ```make
   KUBEBUILDER_ASSETS="$(shell $(ENVTEST) -v debug --bin-dir $(LOCALBIN) use $(ENVTEST_K8S_VERSION) -p path)"
   ```

3. Run test either with the `go test` or the `ginkgo` executor
   ```shell
   go test ./test/functional/...
   ```
   ```shell
   ginkgo ./test/functional/...
   ```

### Debug in vscode

It is possible insert breakpoints and do step by step execution both in the
controller code and in the test code when you run the test with EnvTest.
Here is an example launch configuration for vscode:
```json
{
    "version": "0.2.0",
    "configurations": [

        {
            "name": "Launch Package",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}/test/functional/suite_test.go",
            "env": {
                "KUBEBUILDER_ASSETS": "${workspaceFolder}/bin/k8s/1.25.0-linux-amd64",
                "OPERATOR_TEMPLATES": "${workspaceFolder}/templates"
            },
            "args": [
                "-ginkgo.v",
                "-ginkgo.focus",
                "<name of the test case to execute>"
            ]
        }
    ]
}
```

Note that you at least needs to run the test with the make target to get the
binaries downloaded to the local ```./bin``` folder.

### Debug with delve
You can also use the command line debugger to debug the envtest execution:
```shell
KUBEBUILDER_ASSETS=$(pwd)/bin/k8s/1.26.1-linux-amd64 OPERATOR_TEMPLATES=$(pwd)/templates dlv test ./test/functional/...
```

### CI

If you have a make target defined based on the above steps then you can simply
add a prow job to run that make target for every PR. See for the
[functional job](https://github.com/openshift/release/blob/44ccbf4173f736cfb38e0268280efa02954761ee/ci-operator/config/openstack-k8s-operators/nova-operator/openstack-k8s-operators-nova-operator-master.yaml#L46-L49) in nova-operator.

## Best practices

**Tips and tricks for improving Ginkgo tests**

Tips and tricks for improving the efficiency and organization of your Ginkgo tests:

1. To avoid duplicating general test setup code, utilize Ginkgo's global `BeforeEach` and `AfterEach` functions. These should be placed in the [top-level suite](https://github.com/openstack-k8s-operators/nova-operator/blob/main/test/functional/suite_test.go#L237) to ensure consistent setup and teardown across all tests. This BeforeEach can be combined with [`BeforeEach`](https://github.com/openstack-k8s-operators/nova-operator/blob/main/test/functional/nova_multicell_test.go#L100) in any test lvl. More `BeforeEach` and `Context` [here](https://onsi.github.io/ginkgo/#shared-behaviors).

2. When using envtest, create a unique namespace for each test run. This is necessary because namespaces cannot be deleted in a locally running envtest. For more information, refer to the [Kubebuilder documentation](https://book.kubebuilder.io/reference/envtest.html#namespace-usage-limitation) on namespace usage limitation.

3. Use Ginkgo's table entry functionality to reduce the number of individual tests. By consolidating multiple test cases into a table, you can streamline your test suite and improve readability. Remember that if you want to use anything that is initialized in `BeforeEach`, ginkgo doesn’t know about it during [Spec traversing](https://github.com/onsi/ginkgo/issues/378). This augmenting issue can be avoided by similar patterns like [here](https://github.com/openstack-k8s-operators/nova-operator/blob/e07bd0cfbd9df09a208b64a97a943b752f416b1e/test/functional/nova_reconfiguration_test.go#L367).

4. Please avoid using hardcoded paths; instead, try adopting a similar approach as demonstrated in this [example](https://github.com/openstack-k8s-operators/nova-operator/commit/3ab489b1f59f7bf9fe6efdc72fbe058c65732318).

5. To divide tests into smaller, more logical components, you can utilize `By`  [statements](https://onsi.github.io/ginkgo/#documenting-complex-specs-by).

6. To divide tests into more logical segments and conveniently run only a portion of them, you can utilize [labels](https://onsi.github.io/ginkgo/#spec-labels).

### operator-lint
The [operator-lint](https://github.com/gibizer/operator-lint) static checker
enforces some EnvTest related
[rules](https://github.com/gibizer/operator-lint/tree/main/linters/envtest/T001).
So run operator-lint on your project to catch common mistakes.

## References
* [Kubebuilder envtest doc](https://book.kubebuilder.io/reference/envtest.html)
* [Kubebuilder envtest example](https://book.kubebuilder.io/cronjob-tutorial/writing-tests.html)
* [Gomega (the assert library)](https://github.com/onsi/gomega)
* [Ginkgo (the BDD test framework)](https://github.com/onsi/ginkgo)
* [The nova-operator as the example test suite](https://github.com/openstack-k8s-operators/nova-operator/tree/main/test/functional)
