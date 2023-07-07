# Webhooks

Webhooks allow validation and/or defaulting/mutating logic to be added to the Kubernetes API for the creation, updating and deletion of custom resources (CRs).  This enables us to execute validation/mutation against an incoming CR _before_ it is admitted to the reconcile loop of its associated operator.  In this manner, in the case of validation, we can reject improper CR specifications immediately and report that feedback to the user (as opposed to having the reconcile loop within the associated operator detect the problem and surface the error in the status section of the CR, which is a slower feedback mechanism).  Mutation webhooks can be used to set defaults and keep such considerations out of the associated operator's reconcile loop, which makes for a cleaner division of data and its processing logic.

General information about Kubernetes/OpenShift webhooks can be found here:

* https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
* https://docs.openshift.com/container-platform/4.12//operators/understanding/olm/olm-webhooks.html

## Why did we introduce webhooks?

The initial injection of webhooks into our operators was done to achieve greater flexibility with container image defaults -- that is, for default OpenStack service images, not for the operators themselves.  By using mutating/defaulting webhooks, we are able to remove hard-coded `kubebuilder` annotation defaults from our CRD Golang types and instead provide them via environment variables ([example](https://github.com/openstack-k8s-operators/cinder-operator/blob/main/config/default/manager_default_images.yaml)).  These environment variables are read by the operator controller-manager during its initialization ([example](https://github.com/openstack-k8s-operators/cinder-operator/blob/feac84f5479a33a708ed94d5deedf9366b328038/main.go#L163-L171)) and are then available to the controller-manager's mutating/defaulting webhooks during CR creation, where they can be applied if the user has not included an explicit image in the CR definition ([example](https://github.com/openstack-k8s-operators/cinder-operator/blob/feac84f5479a33a708ed94d5deedf9366b328038/api/v1beta1/cinder_webhook.go#L66-L94)).

Why do we need this environment-variable-based flexibility?

1. It allows for OpenStack service container image defaults to be changed in a live deployment _without_ having to respin an operator build and reinstall the updated operator (which would otherwise have to be done to pick up a `kubebuilder` annotation change).  Instead, the CSV for a live deployment of an operator can be edited to adjust the container image environment variables to point to whatever default is necessary.  This is extremely useful for what are known as "offline" or "air-gapped" or "disconnected" OCP clusters.  These types of clusters do not have external Internet access and would be unable to reach either the upstream or downstream (online) registries to pull the default container images.  Instead, after the operator is initially deployed, the CSV's environment variable declarations can be modified by the cluster administrator to point to images available in an internally-accessible private registry.

2. It makes the operators easier to maintain in a downstream context.  Rather than requiring a specific code patch to modify the CRD Golang structs for downstream builds, the downstream build logic can instead just rebase on every upstream import and override the container image environment variables in the CSVs it uses to build the operator bundles.

3. It makes testing for QE/CI easier.  Similar to #1 and #2, the specific container images required for downstream testing can be easily changed in the CSV.  This avoids requiring rebuilds with different defaults or using CRs with particular downstream container images explicitly specified.

## Adding webhooks to an operator

`operator-sdk` provides convenient scaffolding commands to add webhooks to an operator.  For example:

```bash
cd cinder-operator
operator-sdk create webhook --group cinder --version v1beta1 --kind Cinder --programmatic-validation --defaulting
```

`--programmatic-validation` indicates that you want validation webhooks, while `--defaulting` signals that you want mutating ones.  Using both flags is advisable, even if you do not plan to immediately use both types of webhooks.  The reason for this is that using the command again requires a complete wipe and re-scaffolding of any existing webhook content (done via the `--force` flag, by the way).  So it is better to have both -- even if one is empty scaffolding -- in the interest of not having to re-inject your specific webhook logic for one type of webhook if you later decide that you want the other type of webhook.

Once the webhooks have been scaffolded, you will find them in the `api/<version>` directory.  Examine this file to get an idea of what the scaffolding has provided:

```bash
$ ls api/v1beta1
...
-rw-rw-r--. 1 ocp ocp  4024 Mar 14 11:24 cinder_webhook.go
...
```

There are also additions/changes made to the `config` directory and to `main.go`.  Without going into detail in this doc, the `config` directory will now contain additional YAML to bundle the webhook definitions via operator-sdk such that they are installed alongside the operator via OLM.  The `main.go` changes consist of extra code to start a webserver within the operator controller-manager to serve the endpoints for the validating and mutating webhooks.

It is furthermore recommended to add support for running the operator locally with webhooks.  The pattern for doing so can be seen here: https://github.com/openstack-k8s-operators/cinder-operator/pull/155/files.

### Reusing webhook logic

Both the defaulting and the validation logic of the webhook are implemented in
the service operators but to apply them meaningfully we need to call them
from the openstack-operator as the webhhooks will run there in a real
deployment to default values on the top level (i.e. OpenStackControlPlane) CRD,
to provide valuable feedback to the end user while blocking invalid Spec
values.

To do so we need to follow an API pattern described below.

**Defaulting**

For the example lets assume that the service CRD is called Foo

In the service operator:
```golang
func (r *Foo) Default() {
	r.Spec.Default()
}

func (spec *FooSpec) Default() {
    // Implement the defaulting logic on the Spec struct here
}
```

In the openstack-operator:
```golang
func (r *OpenStackControlPlane) Default() {
	r.DefaultServices()
}

func (r *OpenStackControlPlane) DefaultServices() {
	if r.Spec.Foo.Enabled {
		r.Spec.Foo.Template.Default()
	}
```
So when the webhook runs in the service operator the call chain is:
1. Foo.Default()
2. FooSpec.Default()

When the webhook runs in the openstack-operator the call chain is:
1. OpenStackControlPlane.Default()
2. FooSpec.Default()

Therefore the service operator should not implement any defaulting logic in
Foo.Default but all should be in FooSpec.Default() or in deeper calls

For an implementation example see:
* [nova-operator](https://github.com/openstack-k8s-operators/nova-operator/blob/769bb7a67de3cbc28c84c944301cef9b9bb157cc/api/v1beta1/nova_webhook.go#L64-L103)
* [openstack-operator](https://github.com/openstack-k8s-operators/openstack-operator/blob/21e36c588ab7ff0c0dd441a393d121c96e648d13/apis/core/v1beta1/openstackcontrolplane_webhook.go#L238-L372)


**Validation**

The validation webhook is more complex than the defaulting one as it:
1. Needs to return errors to the caller that can be different depending on
   where the webhook runs
2. Needs to specify the path of the field in the validated structure that
   caused the validation error.
3. Validation has multiple types, validation during create, update, or even
   delete. The update validation has access to both the old and the new struct
   value to be able to detect the change.

The validation webhook needs to return a single StatusError even if it detects
multiple failures. We chose to use an Invalid StatusError that is translated to
a HTTP 422 response code. However within the response there is a cause list
where the validation webhook can describe each error independently by passing
an ErrorList to the NewInvalid call.

Still assume that Foo is a service CRD.

In the service operator
```golang
func (r *Foo) ValidateCreate() error {
    errors := r.Spec.ValidateCreate(field.NewPath("spec"))

    if len(errors) != 0 {
        log.Info("validation failed", "name", r.Name)
        return apierrors.NewInvalid(
            schema.GroupKind{Group: "foo.openstack.org", Kind: "Foo"},
            r.Name, errors)
    }
    return nil
}

func (r *FooSpec) ValidateCreate(basePath *field.Path) field.ErrorList {
    // Implement the create validation here and return a list of errors.
    var errors field.ErrorList
    errors = append(errors,  r.ValidateMyFieldLength(basePath)...)

    return errors
}

// ValidateMyFieldLength just an example validation, it is not part of the
// pattern itself. The pattern stops at the Spec level.
func (r. *FooSpec) ValidateMyFieldLength(basePath *field.Path) field.ErrorList {
    var errors field.ErrorList

    if len(r.myField) > 35 {
        errors = append(
            errors,
            // Use the basePath to specify the absolute field path in the error
            field.Invalid(
                basePath.Child("myField"),
                r.myField,
                "should be shorter than 36 characters",
            )
        )
    }
    return errors
}

func (r *Foo) ValidateUpdate(old runtime.Object) error {
    oldFoo, ok := old.(*Foo)
    if !ok || oldFoo == nil {
        return apierrors.NewInternalError(fmt.Errorf("unable to convert existing object"))
    }

    errors := r.Spec.ValidateUpdate(oldFoo.Spec, field.NewPath("spec"))

    if len(errors) != 0 {
        log.Info("validation failed", "name", r.Name)
        return apierrors.NewInvalid(
            schema.GroupKind{Group: "foo.openstack.org", Kind: "Foo"},
            r.Name, errors)
    }
    return nil
}

func (r *FooSpec) ValidateUpdate(old *FooSpec, basePath *field.Path) field.ErrorList {
    // Implement the create validation here and return a list of errors.
    var errors field.ErrorList
    // You can also reuse specific validation from create validation here if
    // needed
    errors = append(errors,  r.ValidateMyFieldLength(basePath)...)

    return errors
}

```

In the openstack-operator
```golang
func (r *OpenStackControlPlane) ValidateCreate() error {
    var allErrs field.ErrorList
    basePath := field.NewPath("spec")
    if err := r.ValidateCreateServices(basePath); err != nil {
        allErrs = append(allErrs, err...)
    }

    if len(allErrs) != 0 {
        return apierrors.NewInvalid(
            schema.GroupKind{Group: "core.openstack.org", Kind: "OpenStackControlPlane"},
            r.Name, allErrs)
    }

    return nil
}

func (r *OpenStackControlPlane) ValidateCreateServices(basePath *field.Path) field.ErrorList {
    var errors field.ErrorList
    // We do validation that are not Create and Update specific here
    errors := append(errors, r.ValidateServices(basePath)...)
    // But also call the service Spec specific Create validation
    if r.Spec.Foo.Enabled {
        errors = append(errors, r.Spec.Foo.Template.ValidateCreate(basePath.Child("foo").Child("template"))...)
    }
    return errors
}

func (r *OpenStackControlPlane) ValidateUpdate(old runtime.Object) error {
    oldControlPlane, ok := old.(*OpenStackControlPlane)
    if !ok || oldControlPlane == nil {
        return apierrors.NewInternalError(fmt.Errorf("unable to convert existing object"))
    }

    var allErrs field.ErrorList
    basePath := field.NewPath("spec")
    if err := r.ValidateUpdateServices(oldControlPlane.Spec, basePath); err != nil {
        allErrs = append(allErrs, err...)
    }

    if len(allErrs) != 0 {
        return apierrors.NewInvalid(
            schema.GroupKind{Group: "core.openstack.org", Kind: "OpenStackControlPlane"},
            r.Name, allErrs)
    }

    return nil
}

func (r *OpenStackControlPlane) ValidateUpdateServices(old *OpenStackControlPlaneSpec, basePath *field.Path) field.ErrorList {
    var errors field.ErrorList
    // We do validation that are not Create and Update specific here
    errors := append(errors, r.ValidateServices(basePath)...)
    // But also call the service Spec specific Update validation
    if r.Spec.Foo.Enabled {
        errors = append(
            errors,
            r.Spec.Foo.Template.ValidateUpdate(old.Foo.Template, basePath.Child("foo").Child("template"))...)
    }
    return errors
}
```

So when the webhook runs in the service operator for a create the call chain
is:
1. Foo.ValidateCreate()
2. FooSpec.ValidateCreate()
3. FooSpec.ValidateMyFieldLength()

When the webhook runs in the openstack-operator for an update the call chain
is:
1. OpenStackControlPlane.ValidateUpdate()
2. FooSpec.ValidateUpdate()
3. FooSpec.ValidateMyFieldLength()

This validation chaining pattern can be further extended inside the service
operator if the service operator not just implement a top level service CRD but
also sub CRDs.

For an implementation example see:
* [nova-operator](https://github.com/openstack-k8s-operators/nova-operator/blob/769bb7a67de3cbc28c84c944301cef9b9bb157cc/api/v1beta1/nova_webhook.go#L133-L174)
* [openstack-operator](https://github.com/openstack-k8s-operators/openstack-operator/blob/21e36c588ab7ff0c0dd441a393d121c96e648d13/apis/core/v1beta1/openstackcontrolplane_webhook.go#L59-L101)

## Running an operator that has webhooks via OLM

When the associated operator is deployed via OLM, its webhooks are installed in the cluster and cert management is automatically handled as well.  You don't need to do anything differently than you normally would for installing the operator.

### Changing webhook defaults in OLM context

To change the webhook environment variable defaults (those mentioned [here](https://github.com/openstack-k8s-operators/docs/blob/main/webhooks.md#why-did-we-introduce-webhooks)) for an operator that was deployed via OLM, do the following _after_ you have installed the operator and its CSV reaches the `Succeeded` state:

1. Create a patch file to change the desired default

```bash
oc get csv -n openstack-operators -l operators.coreos.com/<OPERATOR_NAME>.openstack-operators -o=jsonpath='{.items[0]}' | jq '(.spec.install.spec.deployments[0].spec.template.spec.containers[1].env[] | select(.name=="<DEFAULT NAME>")) |= (.value="<NEW_DEFAULT_VALUE>")' > default_patch.out
```

Where...
- `<OPERATOR_NAME>` is the name of the operator, for example: `keystone-operator`
- `<DEFAULT_NAME>` is the default you wish to change, for example: `KEYSTONE_API_IMAGE_URL_DEFAULT`
- `<NEW DEFAULT VALUE>` is the new default value you desire, for example: `quay.io/somerepo/someimage`

If you have multiple defaults that you wish to change, add additional `(.spec.install.spec.deployments[0].spec.template.spec.containers[1].env[] | select(.name=="<DEFAULT NAME>")) |= (.value="<NEW_DEFAULT_VALUE>")` pipes to the `jq` command for each default.

2. Apply the patch file against the CSV

```bash
oc patch csv -n openstack-operators $(oc get csv -n openstack-operators -l operators.coreos.com/<OPERATOR_NAME>.openstack-operators -o jsonpath='{.items[0].metadata.name}') --type=merge --patch-file=default_patch.out
```

This will cause the operator's controller-manager pod to be redeployed with your new environment variable defaults.

## Locally running an operator that has webhooks

If you would like to run the operator locally, extra steps must be taken to either disable the webhooks completely or to enable them to run outside of an OLM context.

Before following either section below, make sure you're pointing to a valid `KUBECONFIG` or have otherwise used `oc` to log into your cluster.

### Disabling webhooks

You can disable the webhooks if you don't need them for your local dev/testing purposes.  The procedure is as follows:

1. If you installed the operator via OLM, remove its webhook definitions from its CSV:

```bash
oc patch csv -n openstack-operators <your operator CSV> --type=json -p="[{'op': 'remove', 'path': '/spec/webhookdefinitions'}]"
```

2. Run the operator locally with the webhook server disabled:

```bash
cd <your operator root dir>
ENABLE_WEBHOOKS=false GOWORK= OPERATOR_TEMPLATES=./templates make run
```

### Using webhooks locally

Webhooks can be used outside of an OLM context if you want or need them.  However, using webhooks locally is non-trivial and is best handled by adding something like [this](https://github.com/openstack-k8s-operators/cinder-operator/pull/155/files) to the operator itself (as mentioned in the [Adding webhooks to an operator](#adding-webhooks-to-an-operator) section above).  This doc will only cover that use case for the time being.  Thus, do the following to use webhooks locally where the aforementioned support is present:

1. First, if you installed the operator via OLM, remove its webhook definitions from its CSV:

```bash
oc patch csv -n openstack-operators <your operator CSV> --type=json -p="[{'op': 'remove', 'path': '/spec/webhookdefinitions'}]"
```

2. Now execute the `make` target to run the operator locally with webhooks enabled:

```bash
cd <your operator root dir>
GOWORK= OPERATOR_TEMPLATES=./templates make run-with-webhook
```

This `make` command will:

1. Create self-signed certs for the webhooks and create `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` resources within the cluster that use the certs.  The operator itself also knows where the find the certs in a default location on your local host (`/tmp/k8s-webhook-server/serving-certs`) and will use them when it starts running.
2. Find the OpenShift SDN gateway IP for your CRC cluster.  If you are not using CRC, you will need to provide `CRC_IP=<OpenShift SDN gateway IP>` to the aforementioned `make` command!
3. Inject the `CRC_IP` into the `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` resources so that they point the webhook server running in your local operator controller-manager process.
4. Open the local host's firewall to allow port 9443 traffic (the port used by the webhooks).  This is done in the `libvirt` `firewall-zone`.  _If your local cluster is running in a different zone, you will have to manually add a firewall rule for port 9443 TCP traffic yourself!_
5. Finally, run the actual operator controller-manager and the webhook server.

You should now be able to create CRs for the associated operator and have the webhook logic execute as expected.
**NOTE:** If you want to switch back to using the OLM-deployed version of your operator, you will need to manually `oc delete` the `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` resources created by this `make` command!
