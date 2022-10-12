
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

## Patching vs Updating CR Status
Our CRDs have `Spec` and `Status` subresources. The `Spec` represents the
input to the reconciler. The reconciler treats it as a read-only struct. On
the other hand, the reconciler actively updates the `Status` subresource to
provide information to the outside world. The `Status` is treated as
read-only outside of the reconciler. K8s uses the generation concept to ensure
that parallel writes to the same resource result in a consistent state.
However, generation only exists on the CRD level and not on the subresource
level. So while the usage of `Spec` and `Status` makes these two subresources
independent and race-free, an external update to `Spec` can "race" with an
internal update to `Status` on the generation on the CRD level. This means the
controller can get a `Conflict` response from k8s when trying to update the
`Status` of the CR. In our current operator implementations, such conflict
causes a new reconciliation to start with a fresh copy of the CR. To minimize
such re-reconciliation attempts the controller should only update the `Status`
subresource if some status data is changed. The `Helper` in `lib-common`
tracks the changes on the CR instance and can tell if something is changed. It
also makes it possible to use `Patch()` over `Update()` to limit the amount of
data sent to the API server.

The basic usage pattern of the Helper is the following:

1. At the start of the `Reconcile()` call query the state of the CR from the
k8s and instantiate a new `Helper` with it. The `Helper` will copy the actual
state of the CR internally.

2. Modify the CR instance during the `Reconcile()` call

3. At the end of the `Reconcile()` use `Helper.SetAfter()` to provide the new
state of the CR to the helper. Then use `Helper.GetChanges()` to see if there
was any changes and if so you can use the controller-runtime's
`Client.MergeFrom(Helper.GetBeforeObject())` to generate the patch request that
can be used in the `Reconciler.Status().Patch()` to do the minimal update the
`Status` subresource.
