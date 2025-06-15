# OpenStackVersion

## Overview

This document describes how to use the OpenStackVersion
CR to apply a minor updates to a deployed OpenStack
installation (including both Controlplane and Dataplane
Services)

## Requirements

 * Use of OLM (Operator Lifecycle Manager) is assumed
 * The CSV (ClusterServiceVersion) for the openstack-operator
   is used as the primary resource for versioning and the
   associated ENV variables for OpenStack service containers.
   These are typically prefixed with RELATED\_IMAGE...
   The CSV defines the version.
 * OpenStack operators are always upgraded first via OLM. This
   can be either via manual or automatic. Upgrading operators
   may trigger OpenStackVersion to provide access to a new
   available version.
 * And ENV called OPENSTACK\_RELEASE\_VERSION must be set in the CSV
   for openstack-operator.  This controls the 'availableVersion'
   of OpenStackVersion.
   

## General Update Workflow

OpenStack is installed by following the normal installation procedure. When
an OpenStackControlplane resource is created in any given namespace an
OpenStackVersion resource will be immediately created when the OpenStackControlPlane
is reconciled (1 OpenStackVersion per namespace).

When a new set of Operators is installed via OLM (either manually or automatically)
the OpenStackVersion resources will reconcile and a new 'AvailableVersion' will
be set on each CR.

In order to apply an update an administrator sets the 'TargetVersion' to the
same value as the 'AvailableVersion' on the OpenStackVersion resources. This
triggers the OpenStackVersion to reconcile and will update the images
on the associated OpenStackControlplane and OpenStackDataplane resources in
that namespace.

```shell
$ oc get openstackversion
NAME              TARGET VERSION   AVAILABLE VERSION    DEPLOYED VERSION
openstack         1.0.0            1.0.1                1.0.0
```

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackVersion
metadata:
  labels:
  name: openstack
spec:
  targetVersion: 1.0.1
```

## Custom images for Cinder Volume
In order to allow administrators to configure customized Cinder volume backends
with certified 3rd party vendor containerImages for any of the backends, 
administrators will be allowed to explicitly specify those images overriding the
default containerImage during automatic updates. This is a manual operation 
because there is no way to fully automate the selection of those images at this
time. To configure a Cinder volume backend named 'pure' with a different image 
during the updates, the administrator can add the name 'pure' to the
'cinderVolumeImages' map within the 'customContainerImages' section of
the OpenStackVersion CR and set a custom image to be used for that volume backend.
Multiple named Cinder volume backends may be configured in this manner.

NOTE: if the backend does not require a custom container the default cinder
volume container is used automatically. This means if you are only using
backends that requires the default cinder volume container no
edits to the OpenStackVersion resource are needed.

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackVersion
metadata:
  labels:
  name: openstack
spec:
  targetVersion: 1.0.1
  customContainerImages:
    cinderVolumeImages:
      default: quay.io/podified-antelope-centos9/openstack-cinder-volume@sha256:886942a49a89be8ee12b3b6c220cbea3aead4ee3d9f18f97757eb7d795f241c6
      pure: <custom image URL location>
```

## Custom images for Manila Share

Similar to the Cinder example above certified Manila Share images
may be customized by updating the 'manilaShareImages' map
within the 'customContainerImages' section of the OpenStackVersion
CR and then set to a custom container image for the named backend.

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackVersion
metadata:
  labels:
  name: openstack
spec:
  targetVersion: 1.0.1
  customContainerImages:
    manilaShareImages:
      default: quay.io/podified-antelope-centos9/openstack-manila-share@sha256:55c6acf35d6a4e26228a49d5dcc86a8e0ca54d32b55e3465535b2aa0111f76e2
      custom: <custom image URL location>
```

## Custom Images for Neutron ML2

TBD. There is currently no formal interface to deploy custom ml2 drivers
for neutron other than OVN. If such an interface evolves we can
add a mechanism to customize any container images associated with
that interface to OpenStackVersion.

## Custom images for other OpenStack services

In general administrators should not need to customize containerImages
for any other OpenStack services images. The exception would be
the need to apply a hotfix. In order to accommodate a hotfixed or
patched containerImage you can update the 'customContainerImages'
struct on the OpenStackVersion with the customized image.
For example if you need to hotfix the Glance API service container image
you would add the following update to the glanceAPIImage field within
the customContainerImages section:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackVersion
metadata:
  labels:
  name: openstack
spec:
  targetVersion: 1.0.1
  customContainerImages:
    glanceAPIImage: <custom image URL location>
```

## Ordering of service updates: normal vs minor updates

Container image updates currently occur in parallel when updating
custom images of the OpenStackVersion resource. On the ControlPlane
this means that any image customizations on the OpenStackVersion
resource will be applied immediately. And on the Dataplane
those updates will be applied during the next 'deployment'.

The exception to this is if targetVersion is changed thus causing a minor update
sequence to occur. If a minor update is triggered a special sequence is
followed until all ControlPlane and Dataplane components have their deployedVersion
set to the new targetVersion value. The minor update sequence is as follows:

1. OVN controller updates. The OVN controllers on both the ControlPlane and DataplaneNodesets
   are updated. These services can be updated in parallel however the ControlPlane components
   will begin automatically immediately after the targetVersion is edited. DataplaneNodeset
   updates currently are required to be executed manually.
2. All the remaining services on ControlPlane are updated to the new service image containers.
3. The rest of the Dataplane service containers are updated. DataplaneNodesets updates
   are currently required to be executed manually.

TODO: If a situation arises where we need to update one service before
another we can use OpenStackVersion to implement this in the future which
may require the addition of more conditional steps to orchestrate
the update at a high level.

## Things currently not supported

 * Rolling back to a prior openstack version directly via OpenStackVersion.
   A rollback would require the administrator to install an old CSV,
   and patch various CRs to update the containerImages among other things.
   While a rollback may be allowed for development testing in production
   rollbacks are not recommended/supported.
 * Major upgrades are not targeted at this time. However the OpenStackVersion
   CR may be useful as a means to help drive those as well.

## Managing OpenStack Service Version Incompatibilities

OpenStack services can exhibit incompatibilities between versions that require
careful handling during updates to prevent services from entering a
`CrashLoopBackOff` state. An example is `Glance`, which changed its default
deployment model from `httpd+ProxyPass` to `httpd+WSGI` between feature releases.
In brownfield environments, this creates a challenge: transitioning `Glance` from
one deployment model to another without breaking the existing ControlPlane's
ability to reach a `Ready` state.

The solution addresses two critical aspects:

1. **Backward Compatibility in Service Operators**: when breaking changes are
   introduced, service operators must maintain the ability to orchestrate both
   legacy and current deployment models simultaneously.

2. **Informed Decision Making**: the openstack-operator must provide a
   mechanism for building custom resources (CRs) with sufficient information to
   enable services to make informed deployment decisions.

### Implementation Pattern

The solution that addresses this problem leverages an `annotation-based` pattern
centered on the `OpenStackVersion` Kubernetes CR. This CR triggers updates and
provides the necessary data for version transitions.
A new `ServiceDefaults` field has been added to the `OpenStackVersion` Status:

```golang
// ServiceDefaults - struct that contains defaults for OSP services that can
// change over time but are associated with a specific OpenStack release version
type ServiceDefaults struct {
	GlanceWsgi *string `json:"glanceWsgi,omitempty"`
}
```

`ServiceDefaults` values vary across `discovered releases`.
Each value in `.Status.AvailableVersion` can have its own associated defaults.
Rather than relying on specific version numbers, this mechanism allows
arbitrary values to be associated with particular OpenStack releases, providing
flexibility in version management.
During CR reconciliation, the service defaults are used to `annotate` the
resulting service.
Service operators then process these annotations to make deployment decisions.
The service operator serves as the final component in this chain, interpreting
annotations and implementing the appropriate deployment strategy based on their
semantics.
This approach ensures smooth transitions between incompatible service versions
while maintaining system stability and operator control over deployment models.

### Example

The diagram below shows how `Glance` has two different annotation values defined
in the `serviceDefaults` `OpenStackVersion` Status field.
Based on the values, which is translated into a Glance top-level CR annotation,
`glance-operator` orchstrates the deployment of the underlying `GlanceAPI`
resources accordingly.

```
+---------------------------------------+      +---------------------------------------------------+       +-------------------------------------------------+
|      +----------------------+         |      |        +----------------------+                   |       |                                                 |
|      |  openstack-version  | -------------------------| openstack-controller |                   |       |    +-------------------+                        |
|      +----------------------+         |      |        +----------------------+                   |       |    |   glance-operator |                        |
|        |                              |      |            |                                      |       |    +-------------------+                        |
|        |                              |      |            | (annotate Glance CR)                 |       |      |                                          |
|        |-> (FR2) [GlanceWSGI: false]-------------------------> [glance.openstack.org/wsgi: false]------->|      +-> if (glance.openstack.org/wsgi) == true |
|        |                              |      |            |                                      |       |          then                                   |
|        |                              |      |            |                                      |       |             deploy: httpd+wsgi                  |
|        |                              |      |            |                                      |       |          else                                   |
|        |                              |      |            | (annotate Glance CR)                 |       |             deploy: httpd+proxypass             |
|        |-> (FR3) [GlanceWSGI: true]--------------------------> [glance.openstack.org/wsgi: true]-------->|                                                 |
|                                       |      |            | (minor updates)                      |       |                                                 |
|                                       |      |                                                   |       +-------------------------------------------------+
+---------------------------------------+      +---------------------------------------------------+
```


#### Flow Explanation

1. **ServiceDefaults**: The `OpenstackVersion` object sets for different
   OpenStack releases (FR2, FR3) the corresponding `GlanceWSGI` values in
   `serviceDefaults`

2. **Annotation Propagation**: The openstack-controller reads these defaults
   and applies them as annotations to the Glance CR:

   - **FR2**: `glance.openstack.org/wsgi: false`
   - **FR3**: `glance.openstack.org/wsgi: true`

3. **Deployment Decision**: The `glance-operator` reads the annotation and
   selects the appropriate deployment model:

   - **wsgi: true**: Deploy GlanceAPI with httpd+WSGI
   - **wsgi: false**: Deploy GlanceAPI with httpd+ProxyPass

This pattern enables seamless transitions between deployment models while
maintaining backward compatibility during updates.
