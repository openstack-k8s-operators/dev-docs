# Running a local copy of an operator for development without conflicts

You can run a local copy of an operator for development using commands
like the following in that operator's directory.
```
OPERATOR_TEMPLATES=$PWD/templates make run
```
However, the operator deployed by the meta operator (from when you
ran `make openstack`) will conflict with your local copy. E.g. both
operators might try to resolve the same CR.

To avoid the conflict, scale down the OpenStack operator CSV's deployment
`replicas` to 0 and then scale down the associated service operator deployment
`replicas` to 0 as well.

## Disabling a service

1. Drop the OpenStack operator's CSV's deployment `replicas` to 0

```bash
oc patch csv -n openstack-operators <OpenStack operator CSV> --type json \
  -p="[{"op": "replace", "path": "/spec/install/spec/deployments/0/spec/replicas", "value": "0"}]"
```

2. Drop the service operator's deployment `replicas` to 0

```bash
oc patch deployment -n openstack-operators <service operator deployment> --type json \
  -p="[{"op": "replace", "path": "/spec/replicas", "value": "0"}]"
```

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
