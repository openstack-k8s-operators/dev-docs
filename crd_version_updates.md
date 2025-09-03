# CRD version updates

CRDs in our operators are versioned. As a convention almost all of our APIs have an initial version
of 'v1beta1'. This document is meant to describe the general semantics involved in providing
an upgrade path from a 'v1beta1' to a new API version for example called 'v1beta2'.
There are some general upstream kubernetes/kubebuilder [conversion concepts](https://book.kubebuilder.io/multiversion-tutorial/conversion-concepts.html) that can be referenced for best practices. This document is
more about how the use of those recommended best practices can be applied to operators within
openstack-k8s-operators.

# CRD storage and size concerns

Each CRD definition contains all versions in a single file. So for example the CRD definition for
keystoneapi would contain version information for both the v1beta1 and v1beta2 API definitions.
When it comes to size concerns within services operators these multiple versions don't matter much as
we aren't anywhere near the storage limits in this case. For services that roll up into OpenStackControlplane
we have enough space for about 4 versions at this point. This effectively means we could maintain
an upgrade path from 3 previous versions to a current new version in openstack-operator for that CRD.

# Hub and Spoke details

In reading the upstream docs there is some ambiguity about which API version is the hub and which is the
spoke. As a matter of preference it would be preferred to make the *latest* CRD version the Hub
and any older versions still maintained in the operator codebase the spokes. This would involve
implementing ConvertFrom/ConvertTo interfaces for the older APIs directories.

# Webhooks

For the purpose of conversion a new type of webhook will be used called a conversion webhook. You
can use kubebuilder/operator-sdk to help scaffold these but for the most part it involves implementing
the Hub and Spoke interfaces mentioned in the section above. Additionally the CRD definition for
each operator will need to have a new 'webhook' section and certificate manager CA injection
enabled for that API. You can enable these via the config/crd/kustomization.yaml.

NOTE: openstack-operator itself will need special handling to ensure the CRDs for service operators
 are installed correctly with properly named certificates etc. Specifically, there needs to be
 code that ensures the namespace of these new resources corresponds to the namespace the openstack
 operators get installed to ('openstack-operators' by default).

In general it is best to move other types of webhook code for validations or defaulting to the
latest API version. The main.go file for the operator would then setup webhooks only for the latest
API version (you only need to call SetupWebhookWithManager once for the latest API).

Be sure to update any kubebuilder annotations to use the new API version (Example: v1beta2).

# APIs and Controllers

When you generate the new API you only need to create the API itself, not a new controller. Once you
have the conversion webhooks implemented you can set the new version as the 'storage' version by
adding the 'storageversion' annotation to the API struct root object. Example:

```code
//+kubebuilder:storageversion
```

Migrate the old controller and operator code so that it uses the new API version only and thus the
operator code itself only needs to work with the latest version (aside from the conversion code in the api directory itself).

# API references in other Operators

If the CRD that is getting a new version is referenced in any way in other service operators you will
need to update those versions to look for the new API version instead. This is because once the
conversion happens the API version marked as the storage version is what will be stored. Example: if
service operators query KeystoneAPI to ensure the status is complete it would need to be the latest
v1beta2 API version.

# Service Accounts concerns

Most service operators in openstack-k8s-operators have code in them to create restricted service accounts (SAs) used to deploy the openstack services themselves. These service accounts have ownership set on them
to the CRDs themselves and this is versioned. If you are upgrading a CRD which has ownership set
on an old version you will need to patch the SA to the new version and/or recreate the service accounts
as part of the conversion logic in your controller.

NOTE: there is follow up work here to investigate best practices for ownership openstack SAs

# Bumping the OpenStackControlplane CRD

If the CRD version is exposed via the OpenStackControlplane CRD then a new version of OpenStackControlplane will also need to be created. See above section on size concerns. In general the CRD of the OpenStackControlplane works the same as service operators except there is generally no service account concerns.

# Enabling webhooks in openstack-operator

In general most service operators have webhooks disabled when deployed via openstack-operator. If
and when an API conversion is needed it is necessary to enable the service operator webhooks in the
openstack operator's code base. This will involve edits to controllers/operator/openstack\_controller.go
and hack/sync-bindata.sh. After these edits running a 'make bindata' should show the newly activated
webhooks getting staged in the bindata directory for deployment.


# Timing of CRD conversion with regards to OpenStack minor updates

CRD conversions will occur when the API reconciles which can be periodic or immediately when a CRD with
a new version gets modified in anyway. This could cause a concern for a service disruption or outage
within OpenStack in general. Service operators may consider an immediate update of an API deployment
to be acceptable. If there is a need (for any reason) to keep pods, deployments, statefulsets
untouched until a 'minor update window' then service operators must provide custom code to accommodate
this. A recommended approach might be to support both the old and new deployment structures
internally and use the [service defaults](version_updates.md#managing-openstack-service-version-incompatibilities) mechanism to support migrating
to the new deployment default during a minor update window when a new OpenStackVersion targetVersion
is applied.
