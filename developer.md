
# Implementation guidelines
This is a collection of guidelines for developing the operators. The goal
here is to keep the codebase consistent and help spread good patterns
between the developers.

You can also consult with the [OpenShift Operator Best Practices](https://redhat-openshift-ecosystem.github.io/community-operators-prod/best-practices/).

## Logging
When you log from any of the controller implementations use the lib-common
`utils.LogForObject` and `utils.LogErrorForObject` calls. These will make sure
that the log line has enough information to identify the CR being reconciled.
These calls need a lib-common `Helper` object. So in the short time window
before such an object is created at the start of the reconcile call logging
should be done via the logger provided by the `log.FromContext(ctx)` call.

## Clients
The reconcile loop has access to two k8s clients `client` and `kclient`. As a
rule of thumb use the `client` to query, create, and update k8s objects in the
cluster. The `kclient` is used by lib-common in some edge cases but no new
usage of the `kclient` should be introduced if possible.

## External CRD dependencies
When an OpenStack CRD depends on other CRDs outside of its project like when
the Nova CR depends on a `MariaDB` or a `RabbitMqCluster` CR then such
dependencies should be provided via the name of the CR. Names are unique per
`Namespace` and per `Kind` in kubernetes. Our CRDs implicitly assume that a
field taking the name of another CR has a well-defined Kind. I.e. the field
`NovaSpec.APIDatabaseInstance` expects the name of a `MariaDB` Kind. This way
the name provided always identifies 0 or 1 CR instances in the Namespace.

Alternatively, CRs could be selected via labels or annotations. The label or
annotation based approach gives greater flexibility, i.e. we would not need to
encode various details of a CR into its name e.g.
`MariaDB.Name = "cell1-external"` could be expressed with
`MariaDB.labels = {"name":"cell1", "external":true}`. At the same time, it
would increase the complexity of our external interface and would require the
implementation to handle the case when a label selector matches more than one
CR instance.

Using names to pass dependencies does not mean that we cannot or should not use
label-based queries in our internal implementation. For example, the lib-common
database module uses the name of the `MariaDB` CR to formulate the label
selector query to find the `Service` instance created by the `MariaDB` CR.
