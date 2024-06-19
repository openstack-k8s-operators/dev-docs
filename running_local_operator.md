# Running a local copy of an operator for development without conflicts

You can run a local copy of an operator for development using commands
like the following in that operator's directory.
```
OPERATOR_TEMPLATES=$PWD/templates make run
```
However, the operator deployed by the meta operator (from when you
ran `make openstack`) will conflict with your local copy. E.g. both
operators might try to resolve the same CR.

To avoid the conflict, scale down the operator deployed by the meta
operator by either setting its controller-manager pod replicas to 0
or removing the deployment from the CSV entirely so that only your
local operator will resolve its CR.

## Disabling a service within the CSV

1. Backup the operator's CSV in case you want to restore it later:

```
oc get csv -n openstack-operators <your operator CSV> -o json | \
  jq -r 'del(.metadata.generation, .metadata.resourceVersion, .metadata.uid)'  > operator_csv.json
```

2. Either patch the CSV for the operator so that it scales down to 0 controller-manager pod replicas:

```
oc patch csv -n openstack-operators <your operator CSV> --type json \
  -p="[{"op": "replace", "path": "/spec/install/spec/deployments/0/spec/replicas", "value": "0"}]"
```

...or remove the deployment outright:

```
oc patch csv -n openstack-operators <your operator CSV> --type json \
  -p="[{"op": "remove", "path": "/spec/install/spec/deployments/0"}]"
```

3. Then [remove the webhooks](https://github.com/openstack-k8s-operators/docs/blob/main/webhooks.md#disabling-webhooks)

4. (Optional) To restore the original OLM-based operator after you are done with local dev/testing:

```
oc patch csv -n openstack-operators <your operator CSV> --type=merge --patch-file=operator_csv.json
```

## An Alternative Approach Provided by ChatGPT

In Kubernetes, CSV (Cluster Service Version) is a Custom Resource Definition (CRD) that enables the operator to manage the lifecycle of a specific application in a Kubernetes cluster. The CSV defines the deployment strategy, dependencies, and upgrade paths for the application.

To remove a deployment from the CSV, you need to update the CSV file and remove the reference to the deployment. Here are the steps to do so:

1. Use the kubectl get csv command to get the name of the CSV that contains the deployment you want to remove.

2. Use the kubectl edit csv &lt;csv-name&gt; command to open the CSV in an editor. This command will open the YAML file for the CSV in the default editor specified by your system.

3. Locate the spec.install.spec.deployments section in the YAML file. This section contains a list of all the deployments managed by the CSV.

4. Remove the deployment that you want to delete from the list. Save the changes and close the editor.

5. Verify that the deployment has been removed by running the kubectl get deployments command. The deployment should no longer be listed.

Note: Removing a deployment from the CSV does not delete the deployment from the cluster. You need to delete the deployment manually using the kubectl delete deployment &lt;deployment-name&gt; command.

## Allow local running operator to connect to k8s services

If the operator runs local and has to interact with other k8s services in the cluster won't work.
An example is the keystone operator using gopher using the keystone internal api endpoint.

Errors like the following will be logged when running the operator local:

```
2023-10-16T07:54:49-04:00       ERROR   Reconciler error        {"controller": "keystoneendpoint", "controllerGroup": "keystone.openstack.org", "controllerKind": "KeystoneEndpoint", "KeystoneEndpoint": {"name":"barbican-api","namespace":"openstack"}, "namespace": "openstack", "name": "barbican-api", "reconcileID": "0c71d22a-dae6-41b5-9155-8072eb05f704", "error": "Get \"http://keystone-internal.openstack.svc:5000/\": dial tcp: lookup keystone-internal.openstack.svc: no such host"} sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler
```

Running a tool like `kubefwd` locally allows the local running operator to communicate into the remote cluster.
For more insights on the tool check [Kubernetes Port Forwarding for Local Development](https://imti.co/kubernetes-port-forwarding/)
and the tool at https://github.com/txn2/kubefwd. Pre-compiled binaries can be found at https://github.com/txn2/kubefwd/releases .

To run `kubefwd` to allow access to all k8s servies in the `openstack` namespace:

```
$ export KUBECONFIG=~/.kube/config
$ sudo -E kubefwd services -n openstack
```

When the tool is up, run the operator using `make run`.

If `make run` fails with the following error as there is already another service bound to port `8080/tcp`:

```
2023-10-17T10:24:21+02:00       ERROR   controller-runtime.metrics      metrics server failed to listen. You may want to disable the metrics server or use another port if it is due to conflicts        {"error": "error listening on :8080: listen tcp :8080: bind: address already in use"}
sigs.k8s.io/controller-runtime/pkg/metrics.NewListener
        .../go/pkg/mod/sigs.k8s.io/controller-runtime@v0.14.6/pkg/metrics/listener.go:48
sigs.k8s.io/controller-runtime/pkg/manager.New
        .../go/pkg/mod/sigs.k8s.io/controller-runtime@v0.14.6/pkg/manager/manager.go:407
main.main
        .../src/github.com/openstack-k8s-operators/keystone-operator/main.go:77
runtime.main
        /usr/local/go/src/runtime/proc.go:250
2023-10-17T10:24:21+02:00       ERROR   setup   unable to start manager {"error": "error listening on :8080: listen tcp :8080: bind: address already in use"}
main.main
        .../src/github.com/openstack-k8s-operators/keystone-operator/main.go:86
runtime.main
        /usr/local/go/src/runtime/proc.go:250
exit status 1
make: *** [Makefile:137: run] Error 1
```

Validate that the `Makefile` supports passing the `METRICS_PORT`, similar to:

```
diff --git a/Makefile b/Makefile
index deeb5ec..ea19e9d 100644
--- a/Makefile
+++ b/Makefile
@@ -129,10 +129,12 @@ build: generate fmt vet ## Build manager binary.
        go build -o bin/manager main.go

 .PHONY: run
+run: export METRICS_PORT?=8080
+run: export HEALTH_PORT?=8081
 run: export ENABLE_WEBHOOKS?=false
 run: manifests generate fmt vet ## Run a controller from your host.
        /bin/bash hack/clean_local_webhook.sh
-       go run ./main.go
+       go run ./main.go -metrics-bind-address ":$(METRICS_PORT)" -health-probe-bind-address ":$(HEALTH_PORT)"

 .PHONY: docker-build
 docker-build: test ## Build docker image with the manager.
```

Then pass an available port for the `METRICS_PORT` when running the operator:

```
$ METRICS_PORT=8082 make run
```
