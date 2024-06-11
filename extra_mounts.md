# Strategy to provide files to services (extraMounts)

ExtraMounts represent a feature that aims to extend the operators CRs
and provide an interface to inject files to services.

## The problem

- Provide configuration and credentials files for the storage backend.
  The Ceph cluster configuration and keyring files is one such
  example, and we need those files in Cinder service (volume and
  backup), Glance service, and Compute nodes. Another example would be
  certificates for HTTPS requests to the storage array, policy files,
  asset files in case of Horizon, and any other relevant configuration file
  that is required by the service.

- Some Cinder drivers rely on the ability to bind mount directories in
  order to access content (could be data or an executable!) on the
  host.

- Some deployments run out of space in disk because of the local image
  conversion that happens on the create volume from image operation,
  so we need a way to use an NFS share for it.

- Some drivers need to preserve data that they store during runtime
  and that expect to be present when the service is restarted.

## ExtraMounts

- ExtraMounts are the result of data structures and functions used to
  expose Volumes and Mounts to the OpenStack CRDs.

- Can be added to a pod according to the defined Propagation policy.

- Volumes and VolumeMounts can be anything (ConfigMaps, Secrets, NFS
  shares, etc) and [the same interface provided by k8s is
  exposed](https://kubernetes.io/docs/concepts/storage/volumes/)

## ExtraMounts in a CR

The sample
[core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml](https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml)
has the following in the `OpenStackControlPlane` CR so that the
Glance and Cinder volume pods mount the cephx key and Ceph
configuration files in /etc/ceph.

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
spec:
  extraMounts:
    - name: v1
      region: r1
      extraVol:
        - propagation:
          - CinderVolume
          - GlanceAPI
          extraVolType: Ceph
          volumes:
          - name: ceph
            secret:
              name: ceph-conf-files
          mounts:
          - name: ceph
            mountPath: "/etc/ceph"
            readOnly: true
  ...
```

The example above shows how `ExtraMounts` can be added to the top level
`OpenStackControlPlane` CR. The resulting Volumes are propagated to both
`CinderVolumes` Pods and `GlanceAPI` Pods. If no propagation is defined,
volumes are propagated to all the services that implement the `ExtraMounts`
interface. For more information about propagation, see [Propagating volumes
section](https://github.com/openstack-k8s-operators/dev-docs/blob/main/extra_mounts.md#propagating-volumes).
ExtraMounts can be added to the `OpenStackControlPlane` top-level CR, they
can only be local to the service within the same CR, or they can be defined
multiple times within the same CR.
In general we define two different approaches that can be combined:

1. `global`: ExtraMounts are added to the top-level `OpenStackControlPlane` CR
2. `local`: ExtraMounts are added to the local service within the `OpenStackControlPlane` CR

If both `global` and `local` approaches are used, for example:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
spec:
  extraMounts:
    - name: v1
      region: r1
      extraVol:
        - propagation:
          - CinderVolume
          - GlanceAPI
          extraVolType: Ceph
          volumes:
          - name: ceph
            secret:
              name: ceph-conf-files
          mounts:
          - name: ceph
            mountPath: "/etc/ceph"
            readOnly: true
  cinder:
    template:
      extraMounts:
        - name: v2
          region: r1
          extraVol:
              volumes:
              - name: ceph-1
                secret:
                  name: ceph-conf-files-1
              mounts:
              - name: ceph-1
                mountPath: "/etc/ceph-1"
                readOnly: true
      cinderVolumes:
        ceph:
          networkAttachments:
          - storage
          customServiceConfig: | ...
          ...
          ...
```

volumes are the result of the concatenation between global and local ones.
As a result of the above, `Glance` Pods only receive the secret provided by
`ceph-conf-files`, while the `CinderVolume` `ceph` instance receives both
the content of `ceph-conf-files`, and the content of `ceph-conf-files-1`.

**Note**

To keep the example simple, the secrets are mounted to different target
directories: `/etc/ceph` and `/etc/ceph-1`, but using `projected` secrets we
can obtain different results.


## Operators where ExtraMounts are used

- https://github.com/openstack-k8s-operators/cinder-operator
- https://github.com/openstack-k8s-operators/dataplane-operator
- https://github.com/openstack-k8s-operators/glance-operator
- https://github.com/openstack-k8s-operators/manila-operator
- https://github.com/openstack-k8s-operators/neutron-operator
- https://github.com/openstack-k8s-operators/openstack-ansibleee-operator
- https://github.com/openstack-k8s-operators/openstack-operator
- https://github.com/openstack-k8s-operators/horizon-operator

## Volumes/VolumesMounts Data Model

The following is an explanation of the `volMounts` struct in
[modules/storage/storage.go](https://github.com/openstack-k8s-operators/lib-common/blob/main/modules/storage/storage.go).

```
+-------------------------------------------------------------------------+
|   // VolMounts is the data structure used to expose Volumes and Mounts  |
|   // that can be added to a pod according to the defined Propagation    |
|   // policy                                                             |
|                                                                         |
|   type VolMounts struct {                                               |
|                                                                         |
|       // Propagation defines which pod should mount the volume          |
|       Propagation []PropagationType `json:"propagation,omitempty"` ------------> Propagation: Defines which pod should mount the
|                                                                         |        Volume
|       // Label associated to a given extraMount                         |
|       // +kubebuilder:validation:Optional                               |
|       ExtraVolType ExtraVolType `json:"extraVolType,omitempty"` ---------------> ExtraVolType is a label that can be used to improve
|                                                                         |        the logic in the operator (e.g., see the dataplane
|                                                                         |        [operator usage]())
|       // +kubebuilder:validation:Required                               |
|       Volumes []corev1.Volume `json:"volumes"` --------------------------------> Volumes represents a list of corev1.Volume
|                                                                         |
|       // +kubebuilder:validation:Required                               |
|       Mounts []corev1.VolumeMount `json:"mounts"` -----------------------------> Mounts represents a list of corev1.VolumeMounts
|   }                                                                     |
+-------------------------------------------------------------------------+
```

Examples:

- The `ExtraVolType` label can be used in the logic of an operator: 
[dataplane-operator usage](https://github.com/openstack-k8s-operators/dataplane-operator/blob/5ebd8ad49a6b674c930b24f28f3da4656ac088ef/pkg/deployment/deployment.go#L224-L229).
```go
	for _, extraMount := range extraMounts {
		if extraMount.ExtraVolType == "Ceph" {
			haveCephSecret = true
			break
		}
	}
```
- An operator can include `VolMounts` explicitly in
  &lt;Service&gt;APISpec: [openstack-ansibleee-operator usage](https://github.com/openstack-k8s-operators/openstack-ansibleee-operator/blob/afe2f120aab1a0b27d0d70036d00d95c1ad7cdb0/api/v1alpha1/openstack_ansibleee_types.go#L71-L73)
```go
	// +kubebuilder:validation:Optional
	// ExtraMounts containing conf files and credentials
	ExtraMounts []storage.VolMounts `json:"extraMounts,omitempty"`
```
- An operator can extend the struct as per the service requirements: 
[glance-operator usage](https://github.com/openstack-k8s-operators/glance-operator/blob/5a38bd9e82681d456d1dcab00b7c9e7944db6178/api/v1beta1/glance_types.go#L192-L214)
```go
// GlanceExtraVolMounts exposes additional parameters processed by the glance-operator
// and defines the common VolMounts structure provided by the main storage module
type GlanceExtraVolMounts struct {
	// +kubebuilder:validation:Optional
	Name string `json:"name,omitempty"`
	// +kubebuilder:validation:Optional
	Region string `json:"region,omitempty"`
	// +kubebuilder:validation:Required
	VolMounts []storage.VolMounts `json:"extraVol"`
}

// Propagate is a function used to filter VolMounts according to the specified
// PropagationType array
func (g *GlanceExtraVolMounts) Propagate(svc []storage.PropagationType) []storage.VolMounts {

	var vl []storage.VolMounts

	for _, gv := range g.VolMounts {
		vl = append(vl, gv.Propagate(svc)...)
	}

	return vl
}
```

## Propagating Volumes

- External propagation: ability to allocate and mount a volume across
  different operators (Glance, Cinder, Manila, Ansible, Neutron,
  DataPlane etc).

- Internal propagation: ability to propagate an extraVolume within the
  single operator.

For example, in case of Cinder, we can propagate a volume either to
CinderVolume or CinderBackup, as well as to a single CinderVolume backend
instance.
Both Glance and Manila follow the same approach.

```
                     | +--------+ |
                    |  | Cinder | --------------> Global: All entities deployed by the cinder
                   |   +--------+   |                     operator will mount the Volume
                  |                  |
                 |  +--------------+  |
                |   | CinderVolume | -----------> Group: All CinderVolume(s) Pods will mount the
               |    +--------------+    |                Volume
              |                          |
             |        +---------+          |
            |         | volume1 | ---------------> Instance: The extraVolume will be mounted only
           |          +---------+           |                by the Pod associated with "volume1"
                                                             backend: **different extraVolumes can
                                                             be defined for different CinderVolume
                                                             backends**.
```

## How to add ExtraMounts to your operator

- Add the data structure to the API
- Make sure the Spec.ExtraMounts is properly propagated in the main controller
- Add the logic to process the extraMounts in the deployments/statefulsets 
- Integrate it in the meta-operator
  - Update the CRD
  - Pass the parameter through the top-level spec

## Demo

[![asciicast](https://asciinema.org/a/533951.svg)](https://asciinema.org/a/533951)
