# Operator Controller Manager Pod Labels

This document describes the prescribed operator `controller-manager` pod labeling and label-selection practices that we aim to require for the OpenStack K8s project.

## Summary

An operator's `controller-manager` pod is given certain labels by the `operator-sdk` scaffolding command when the command is used to create the first CRD+controller.  For example, via:

```
operator-sdk create api --group keystone --version v1beta1 --kind KeystoneAPI --resource --controller
```

Unfortunately these default labels are provided in such a way that they will collide with other `operator-sdk`-scaffolded operators deployed in an OCP cluster.  See below.

## Default (Improper) Labels and Selectors

The default labels look like so in the definition for the `controller-manager` deployment/pod:

```
# config/manager/manager.yaml

...
labels:
  control-plane: controller-manager
  app.kubernetes.io/name: keystone-operator
  app.kubernetes.io/component: keystone
...
```

YAMLs for a Prometheus `ServiceMonitor` and an RBAC auth proxy `Service` are also added for the `controller-manager` deployment/pod.  They both contain a label selector to associate them with the pod.  By default, these are defined like so:

```
# config/prometheus/monitor.yaml

...
selector:
  matchLabels:
    control-plane: controller-manager
...
```

```
# config/rbac/auth_proxy_service.yaml

...
selector:
  control-plane: controller-manager
...
```

If webhooks have been added to a particular operator, a YAML definition for a webhook `Service` will also be present and will contain a similar label selector to those shown above:

```
# config/webhook/service.yaml

...
selector:
  control-plane: controller-manager
...
```

The `control-plane: controller-manager` label and label selectors are always created by `operator-sdk` scaffolding and do not vary in their key/value.  Thus if two or more `operator-sdk`-scaffolded operators are deployed in a cluster at the same time, they will contain these labels and select each other's `controller-manager` pod in the context of their Prometheus `ServiceMonitor` and RBAC auth proxy `Service` (and perhaps webhook `Service`).  This is undesirable and incorrect.  It can result in errors like this:

```
$ oc kustomize /home/ocp/install_yamls/out/openstack/keystone/cr | oc apply -f -
Error from server (InternalError): error when creating "STDIN": Internal error occurred: failed calling webhook "mkeystone.kb.io": failed to call webhook: Post "https://keystone-operator-controller-manager-service.openstack.svc:443/mutate-keystone-openstack-org-v1beta1-keystone?timeout=10s": x509: certificate is valid for glance-operator-controller-manager-service.openstack, glance-operator-controller-manager-service.openstack.svc, not keystone-operator-controller-manager-service.openstack.svc
```

Because the webhook traffic went to the wrong `controller-manager` pod due to the conflated `control-plane: controller-manager` labels, a cert error has occurred.

## Revised (Proper) Labels and Selectors

While we want to fix the problem noted above, the default "universal" `control-plane: controller-manager` label does have its benefits.  Simply put, it is useful for aggregate commands such as those that would be executed to gather logs and status.  i.e.:

```
oc logs -n openstack -l control-plane=controller-manager
oc get pods -n openstack -l control-plane=controller-manager -o jsonpath='{.items[*].status.conditions[0].status}'
```

With this in mind, we are opting to leave the default `control-plane: controller-manager` label in place, but then change the label selectors used by the Prometheus `ServiceMonitor`, RBAC auth proxy `Service` and webhook `Service`.  The selector we are using instead is a unique label that is actually already added to the `controller-manager` pod by the `operator-sdk` scaffolding itself, namely:

```
app.kubernetes.io/name: <openstack service name>-operator
```

Using this allows us to uniquely and correctly establish connectivity between the various operator `controller-manager` pods and their respective peripheral services mentioned above.  Thus the YAML for the operators' attached services would then look like this:

```
# config/prometheus/monitor.yaml

...
selector:
  matchLabels:
    app.kubernetes.io/name: <openstack service name>-operator
...
```

```
# config/rbac/auth_proxy_service.yaml

...
selector:
  app.kubernetes.io/name: <openstack service name>-operator
...
```

```
# config/webhook/service.yaml

...
selector:
  app.kubernetes.io/name: <openstack service name>-operator
...
```
