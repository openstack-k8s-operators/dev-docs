
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

## Child objects
When a controller creates a child object during reconcilation it needs to
decide what will be the lifecycle of the child. There are two categories:

1. The child can be deleted at any time because the owner can re-create it
without any service or data loss. For these objects, the owner does not need to
set any finalizers. For example, a `ConfigMap` created to hold a generated
service config of a `PlacementAPI` CR can always be re-generated from the
`PlacementAPI.Spec`.

2. The deletion of the child should be controlled as the owner cannot re-create
the child without data, or service loss. In this case, the owner should set a
finalizer on the child object to control the deletion of it. For example, a
`MariaDBDatabase` child is created to represent a DB schema for a
`PlacementAPI` CR should only be deleted when the `PlacementAPI` CR is deleted
to avoid losing the DB schema under a running placement service binary.

If a CR has child objects from category 2. then that CR also needs to insert a
finalizer to itself so that when the CR is deleted it can remove the finalizers
from its children objects so those can be deleted too.

Also the owner needs to set an OwnerReference to the child object it creates
so when the owner is deleted k8s can do a cascade delete tor remove the child
object automatically.
