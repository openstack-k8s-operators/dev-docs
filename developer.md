
# Implementation guidelines
This is a collection of guidelines for developing the operators. The goal
here is to keep the codebase consistent and help spread good patterns
between the developers.

You can also consult with the [OpenShift Operator Best Practices](https://redhat-openshift-ecosystem.github.io/community-operators-prod/best-practices/).

## Logging
When you log from any of the controller implementations ,
make use of "sigs.k8s.io/controller-runtime/pkg/log","github.com/go-logr/logr".
Create a per operator/controller top level GetLogger func:
```golang

// GetLogger returns a logger object with a logging prefix of "controller.name"
// and additional controller context fields
func (r *ControllerStruct) GetLogger(ctx context.Context) logr.Logger {
	return log.FromContext(ctx).WithName("Controllers").WithName(
		"ControllerStructName")
}
```
And that should be used in each individual controller func:
```golang
func (r *ControllerStruct)... Reconcile... {

	Log := r.GetLogger(ctx)
```
An individual, per reconcile function log object logr instantiation will create
a controller unique, race safe and precise logr object.
It can then be used with log.Info and log.Error with a string / err obj. (for
log.Error)  and more name/value pairs as needed: `log.Info(msg string,
keysAndValues`. The context of the controller, reconcile id , etc would be
automatically included in logger output.

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

### Always return after update
For safety we should return from the reconciler after doing an update on the
instance. If we don't the we can get read-after-write consistency errors in the
reconciler.

The reason is that the reconciler is called with an object from a shared
informer. This has a watch and a cache. As soon as you call Update() or Patch()
the watch is going to update the shared informer, and the reconciler is queued
to be called again with the state of the object at this point. i.e. Without any
of the things the reconciler would change on the instance after this update.
When the reconcile function returns we call our defer func, which updates the
object again. At this point there is a race. The shared informer already has a
previous reconcile queued for us. If we get called again before kube-apiserver
has fired the watch from the second update and the informer has updated based
on it, we will be called with old data. i.e. We write new data and then
immediately get called again with a version of the object which doesn't contain
the changes we just wrote. Worse, this is extremely likely to happen in
practise. Given that we're going to be called again anyway once we call update
we might  as well exit after a single update call and wait.
## Self finalizer
There are two reasons to add a finalizer to a CR instance from its reconciler:

1. If the reconciler creates children CR instances with finalizers. See
[#2. in child objects](#Child-objects)

2. If the instance needs specific cleanup actions. For example running a `Job`
when the instance is being deleted. Note that deleting children CRs are
automatic if `OwnerReferece` is set no explict delete is needed there.

## Defaulting structs
When a CRD has an optional struct field the defaulting of that field needs
special care. Both the field with the struct type need a full default value
defined and each individual subfields in the struct needs default value
defined.

Our common example is the `PasswordSelectors` struct.

```golang
type PlacementAPISpec struct {
	//...
	// +kubebuilder:validation:Optional
	// +kubebuilder:default={database: PlacementDatabasePassword, service: PlacementPassword}
	// PasswordSelectors - Selectors to identify the DB and ServiceUser password from the Secret
	PasswordSelectors PasswordSelector `json:"passwordSelectors,omitempty"`
	//...
}

type PasswordSelector struct {
	// +kubebuilder:validation:Optional
	// +kubebuilder:default=PlacementDatabasePassword
	// Database - Selector to get the Database user password from the Secret
	Database string `json:"database"`
	// +kubebuilder:validation:Optional
	// +kubebuilder:default=PlacementPassword
	// Service - Selector to get the service user password from the Secret
	Service string `json:"service"`
}
```
When `passwordSelectors` is not provided in the input then the default defined
for the `PasswordSelectors` field will be used. But when the
`passwordSelectors` field is in the input but only defines the `database` field
but not the `service` then the default defiened at `Service` field will be
used.

## CRD fields with omitempty

Some fields omited from serialization using the `omitempty` marker,
thus sparing time that would have to be used for encoding them.
By marking field with omitempty, it will not be encoded, if it contains
value defined as empty. This value is type dependent, for example `0` for integers,
or nil for pointers.[0]

It is necessary for the operator code to expect this behavior,
and special care must be taken when it comes to omitting pointers. 
All pointer fields marked as `omitempty` must be checked if they are not nil before use.

When used in conjuction with `Optional` kubebuilder validation, the field
has to be assumed to be empty by default and treated as such.
Pointers to booleans and integers can be especially tricky in cases like this.
Checks for nil value must be implemented before the field is used.
Since it may happen that their empty value, that is false or 0, is actually meaningful.
Then the optional field may be set to value of nil pointer.

The `omitempty` should therefore only be used on cases when one deals with
pointers in general, or pointers to structs specifically.
All other types do not benefit from being omitted.

[0] https://pkg.go.dev/encoding/json#Marshal
[1] https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#optional-vs-required
