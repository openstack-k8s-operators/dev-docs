# Backup/Restore Controller Implementation Guide

This document provides implementation details for the controller-based backup/restore design. Use this as a reference when implementing backup/restore support in OpenStack operators.

## Overview

The implementation spans multiple repositories:
- **lib-common**: Shared backup/restore helper functions, constants, and CRD label cache
- **openstack-operator**: OpenStackBackupConfig controller, CRD labels for OpenStackControlPlane, DataPlaneNodeSet, OpenStackVersion
- **mariadb-operator**: CRD labels for GaleraBackup + PVC labeling
- **glance-operator**: CRD labels + PVC labeling
- **infra-operator**: CRD labels for NetConfig, Reservation, IPSet, Topology, BGPConfiguration, DNSData

All repositories use a `backup_restore` branch for this work.

**Implementation Flow:**
1. **CRD labels** declare which types participate in backup/restore
2. **OpenStackBackupConfig controller** labels CR instances and user-provided resources (Secrets, ConfigMaps, NADs)
3. **Operators** label managed resources they create (PVCs, database password secrets)
4. **OADP** backs up all resources, restores selectively using `backup.openstack.org/restore` labels

**Two Labeling Mechanisms:**

| Mechanism | What it labels | How |
|-----------|---------------|-----|
| **CRD labels** (kubebuilder annotations) | CRD metadata | Declared in Go type definitions, applied at CRD install |
| **OpenStackBackupConfig controller** | CR instances + user-provided resources | Reconciliation loop labels Secrets, ConfigMaps, NADs (without ownerReferences) and CR instances (based on CRD label cache) |

## General Implementation Rules

### Code Patterns
- **Use lib-common helpers**: Always check lib-common for existing helpers before creating new functions
  - `util.MergeStringMaps()` for merging labels and annotations
  - `backup.GetRestoreLabels()` for resource instance labels
  - `backup.BuildCRDLabelCache()` for CRD label cache
- **Use label constants**: Always use constants from `backup` package for label keys
  - `backup.BackupRestoreLabel` = `"backup.openstack.org/restore"`
  - `backup.BackupCategoryLabel` = `"backup.openstack.org/category"`
  - `backup.BackupRestoreOrderLabel` = `"backup.openstack.org/restore-order"`
  - Never hardcode label strings
- **Return maps, don't modify in-place**: Functions should return label/annotation maps, not modify objects directly
- **Use CreateOrPatch pattern**: Operators typically use CreateOrPatch or CreateOrUpdate (not direct Create)
- **Merge labels**: Use `util.MergeStringMaps()` to merge backup labels with existing service labels

### Testing Requirements
- **Functional tests required**: All new functions must have functional tests
- **Table-driven tests**: Use table-driven test pattern (lib-common standard)
- **Test edge cases**: Include nil, empty, and error cases in tests
- **Test file naming**: Tests go in `*_test.go` files alongside the implementation

### Helper Usage
Check these lib-common modules before implementing new helpers:
- `modules/common/backup` - Backup/restore labels, constants, and CRD cache
- `modules/common/util` - General utilities, map merging
- `modules/common/labels` - Label generation and selectors
- `modules/common/object` - Object metadata operations
- `modules/common/secret` - Secret creation and management
- `modules/common/pvc` - PVC creation and management

## Phase 1: lib-common Implementation

### Backup Package

The `common/backup` package in lib-common provides shared functionality.

#### Labels and Constants (`common/backup/labels.go`)

```go
package backup

const (
    // Label keys for CRD metadata (declare backup behavior for the type)
    BackupRestoreLabel      = "backup.openstack.org/restore"       // CRD label: "true" means instances participate in backup/restore
    BackupCategoryLabel     = "backup.openstack.org/category"      // CRD & instance label: "controlplane" or "dataplane"
    BackupRestoreOrderLabel = "backup.openstack.org/restore-order" // CRD & instance label: "00"-"60"
)

// GetRestoreLabels returns a map with backup/restore labels for CR instances
func GetRestoreLabels(restoreOrder string, category string) map[string]string
```

#### Restore Order Constants (`common/backup/restore.go`)

```go
package backup

const (
    // Restore order constants (gaps of 10 allow insertion)
    RestoreOrder00 = "00" // Storage foundation - PVCs
    RestoreOrder10 = "10" // Foundation - NADs, Secrets, ConfigMaps
    RestoreOrder20 = "20" // Infrastructure - Issuers, NetConfig, Topology, BGPConfiguration, DNSData, OpenStackVersion, OpenStackBackupConfig
    RestoreOrder30 = "30" // CtlPlane + networking - OpenStackControlPlane, Reservation
    RestoreOrder40 = "40" // Backup config, IP sets, DataPlane services - GaleraBackup, IPSet, DataPlaneService
    RestoreOrder50 = "50" // Manual steps - database restore, resume deployment
    RestoreOrder60 = "60" // DataPlane - DataPlaneNodeSet
)

const (
    CategoryControlPlane = "controlplane"
    CategoryDataPlane    = "dataplane"
)
```

#### CRD Label Cache (`common/backup/cache.go`)

```go
package backup

// BackupConfig holds backup/restore configuration for a CRD
type BackupConfig struct {
    Enabled      bool
    RestoreOrder string
    Category     string
}

// CRDLabelCache maps CRD names to their backup configuration
type CRDLabelCache map[string]BackupConfig

// BuildCRDLabelCache reads all CRDs and caches their backup labels
func BuildCRDLabelCache(ctx context.Context, c client.Client) (CRDLabelCache, error)
```

**Key Points:**
- Cache built lazily on first reconciliation of OpenStackBackupConfig controller
- Uses CRD label constants for consistency
- Works with operator upgrade cycle (CRD changes = operator restart = cache rebuild)

## Phase 2: CRD Labels

### Label Format

Add labels to CRD metadata using kubebuilder annotations on Go type definitions:

```go
// +kubebuilder:metadata:labels=backup.openstack.org/restore=true
// +kubebuilder:metadata:labels=backup.openstack.org/category=controlplane
// +kubebuilder:metadata:labels=backup.openstack.org/restore-order=30
type OpenStackControlPlane struct {
```

After modifying type definitions, run `make generate manifests` to update generated CRD YAML files.

### OpenStack Operator CRDs

1. **OpenStackControlPlane** (`api/core/v1beta1/openstackcontrolplane_types.go`)
   - `backup-restore=true`, `category=controlplane`, `order=30`

2. **OpenStackDataPlaneNodeSet** (`api/dataplane/v1beta1/openstackdataplanenodeset_types.go`)
   - `backup-restore=true`, `category=dataplane`, `order=60`

3. **OpenStackVersion** (`api/core/v1beta1/openstackversion_types.go`)
   - `backup-restore=true`, `category=controlplane`, `order=20`

4. **OpenStackBackupConfig** (`api/backup/v1beta1/openstackbackupconfig_types.go`)
   - `backup-restore=true`, `category=controlplane`, `order=20`

5. **OpenStackDataPlaneService** (`api/dataplane/v1beta1/openstackdataplaneservice_types.go`)
   - `backup-restore=true`, `category=dataplane`, `order=40`
   - Custom user-created services; default services are recreated by NodeSet controller

### MariaDB Operator CRDs

1. **GaleraBackup** (`api/v1beta1/galerabackup_types.go`)
   - `backup-restore=true`, `category=controlplane`, `order=40`

**Not labeled:** MariaDBDatabase and MariaDBAccount CRDs do not have backup/restore labels. These CRs are recreated by service operators during reconciliation (via `EnsureMariaDBAccount`), which also generates new password secrets. Database SQL data is restored separately via the Galera restore process (order 50).

### Infra Operator CRDs

1. **NetConfig** (`apis/network/v1beta1/netconfig_types.go`)
   - `backup-restore=true`, `category=dataplane`, `order=20`

2. **Topology** (`apis/topology/v1beta1/topology_types.go`)
   - `backup-restore=true`, `category=dataplane`, `order=20`

3. **BGPConfiguration** (`apis/network/v1beta1/bgpconfiguration_types.go`)
   - `backup-restore=true`, `category=dataplane`, `order=20`

4. **DNSData** (`apis/network/v1beta1/dnsdata_types.go`)
   - `backup-restore=true`, `category=dataplane`, `order=20`

5. **Reservation** (`apis/network/v1beta1/reservation_types.go`)
   - `backup-restore=true`, `category=dataplane`, `order=30`
   - Requires NetConfig (order 20) to exist first

6. **IPSet** (`apis/network/v1beta1/ipset_types.go`)
   - `backup-restore=true`, `category=dataplane`, `order=40`
   - Requires Reservation (order 30) to exist first

7. **InstanceHa** (`apis/instanceha/v1beta1/instanceha_types.go`)
   - `backup-restore=true`, `category=controlplane`, `order=20`
   - **Restored with `spec.disabled: "True"`** via resource modifier to prevent fencing of EDPM nodes before they reconnect to the restored control plane
   - Operator must manually re-enable (`spec.disabled: "False"`) after verifying EDPM connectivity

### CRDs NOT Included

**Service operator CRDs** (GlanceAPI, NovaAPI, KeystoneAPI, CinderAPI, NeutronAPI, etc.) do NOT have backup-restore labels. These CRs are operator-managed - after restoring OpenStackControlPlane (order 30), the openstack-operator reconciles and recreates all service CRs. This avoids version mismatches and configuration drift.

### Verify CRD Labels

```bash
# List all CRDs with backup-restore labels
oc get crd -l backup.openstack.org/restore=true

# Check specific CRD
oc get crd openstackcontrolplanes.core.openstack.org -o jsonpath='{.metadata.labels}'

# Filter by category
oc get crd -l backup.openstack.org/restore=true -l backup.openstack.org/category=controlplane
oc get crd -l backup.openstack.org/restore=true -l backup.openstack.org/category=dataplane
```

## Phase 3: OpenStackBackupConfig Controller

The OpenStackBackupConfig controller is the central component that labels resources for backup/restore. It replaces the webhook-based approach with a simpler, centralized controller.

### OpenStackBackupConfig API

```go
// OpenStackBackupConfigSpec defines the desired state of OpenStackBackupConfig.
type OpenStackBackupConfigSpec struct {
    // DefaultRestoreOrder is the restore order assigned to user-provided resources
    // +kubebuilder:default="10"
    DefaultRestoreOrder string `json:"defaultRestoreOrder,omitempty"`

    // Secrets configuration for backup labeling
    // Defaults: Excludes service-cert and osdp-service labeled secrets
    // +kubebuilder:default={enabled:true,excludeLabelKeys:{"service-cert","osdp-service"}}
    Secrets ResourceBackupConfig `json:"secrets,omitempty"`

    // ConfigMaps configuration for backup labeling
    // Defaults: Excludes kube-root-ca.crt and openshift-service-ca.crt
    // +kubebuilder:default={enabled:true,excludeNames:{"kube-root-ca.crt","openshift-service-ca.crt"}}
    ConfigMaps ResourceBackupConfig `json:"configMaps,omitempty"`

    // NetworkAttachmentDefinitions configuration for backup labeling
    // +kubebuilder:default={enabled:true}
    NetworkAttachmentDefinitions ResourceBackupConfig `json:"networkAttachmentDefinitions,omitempty"`
}

// ResourceBackupConfig defines backup labeling rules for a resource type
type ResourceBackupConfig struct {
    Enabled              bool              `json:"enabled,omitempty"`
    ExcludeLabelKeys     []string          `json:"excludeLabelKeys,omitempty"`
    ExcludeNames         []string          `json:"excludeNames,omitempty"`
    IncludeLabelSelector map[string]string `json:"includeLabelSelector,omitempty"`
}
```

### Controller Logic

The controller (`internal/controller/backup/openstackbackupconfig_controller.go`) does four things on each reconciliation:

1. **Label Secrets** - Lists all Secrets in the target namespace, labels user-provided ones (no ownerReferences) with restore order 10
2. **Label ConfigMaps** - Same for ConfigMaps
3. **Label NADs** - Same for NetworkAttachmentDefinitions
4. **Label CR Instances** - Builds a CRD label cache, iterates all CRDs with `backup-restore=true`, lists instances, labels them with the order/category from the CRD

**Resource filtering:**
- Only resources without ownerReferences are labeled (user-provided)
- Resources matching `excludeLabelKeys` are skipped (e.g., `service-cert` secrets)
- Resources matching `excludeNames` are skipped (e.g., `kube-root-ca.crt`)
- Resources already labeled with correct values are skipped (idempotent)

**Watch triggers:**
- The controller watches Secrets, ConfigMaps, and NADs - when any are created/updated, it reconciles
- This ensures newly created user resources are labeled promptly

### Example OpenStackBackupConfig CR

```yaml
apiVersion: backup.openstack.org/v1beta1
kind: OpenStackBackupConfig
metadata:
  name: backup-config
  namespace: openstack
spec:
  defaultRestoreOrder: "10"
  secrets:
    enabled: true
    excludeLabelKeys:
    - service-cert
    - osdp-service
  configMaps:
    enabled: true
    excludeNames:
    - kube-root-ca.crt
    - openshift-service-ca.crt
  networkAttachmentDefinitions:
    enabled: true
```

### RBAC

The controller requires permissions to:
- Get/list/watch/update/patch Secrets, ConfigMaps, NADs
- Get/list/watch CRDs (for building the label cache)
- Get/list/watch/update/patch resources across all `*.openstack.org` API groups (for labeling CR instances)

Each `*.openstack.org` group is listed explicitly in RBAC annotations since Kubernetes does not support wildcard group patterns.

## Phase 4: Operator-Managed Resource Labeling

This phase covers labeling resources created by operator controllers. These resources have ownerReferences but still need backup labels because they contain data that must be preserved.

### PVC Labeling

Operators label PVCs they create with restore order 00 (storage foundation - restored first).

```go
import (
    "github.com/openstack-k8s-operators/lib-common/modules/common/backup"
    "github.com/openstack-k8s-operators/lib-common/modules/common/util"
)

pvcDef := &corev1.PersistentVolumeClaim{
    ObjectMeta: metav1.ObjectMeta{
        Name:      fmt.Sprintf("%s-pvc", instance.Name),
        Namespace: instance.Namespace,
        Labels: util.MergeStringMaps(
            map[string]string{"app": "glance"},
            backup.GetRestoreLabels(backup.RestoreOrder00, ""),
        ),
    },
    // ... PVC spec
}
```

### Database Password Secret Labeling

MariaDB operator labels database password secrets with restore order 10.

### Operator-Managed Resource Summary

| Operator | Resource Type | Restore Order | Notes |
|----------|--------------|---------------|-------|
| **glance-operator** | PVC | 00 | Storage for Glance images |
| **mariadb-operator** | PVC | 00 | Database storage |
| **mariadb-operator** | Secret (password) | 10 | Database user credentials |

## Phase 5: OADP Integration

### Two-Backup Approach

We use two separate OADP backups:

1. **backup-openstack-pvcs** - Contains ONLY PVCs with `backup.openstack.org/backup: "true"` label
   - Filtering done at backup time via labelSelector
   - Uses CSI volume snapshots

2. **backup-openstack-resources** - Contains everything EXCEPT PVCs
   - No filtering at backup time (backs up all resources)
   - Restore uses labelSelector to filter by `backup.openstack.org/restore-order`

### Restore Order Summary

| Order | Resources | Restore Method | Notes |
|-------|-----------|----------------|-------|
| 00 | PVCs | OADP Restore CR | Storage foundation, CSI snapshots |
| 10 | Secrets, ConfigMaps, NADs | OADP Restore CR | User-provided resources |
| 20 | OpenStackVersion, OpenStackBackupConfig, Issuers, NetConfig, Topology, BGPConfiguration, DNSData, InstanceHa | OADP Restore CR | Infrastructure base (InstanceHa restored with `spec.disabled: True`) |
| 30 | OpenStackControlPlane, Reservation | OADP Restore CR | With `deployment-stage: infrastructure-only` annotation |
| 40 | GaleraBackup, IPSet, DataPlaneService | OADP Restore CR | Backup config, IP sets, custom DataPlane services |
| 50 | Database restore, resume deployment | **Manual/Playbook** | GaleraRestore CRs, remove staged annotation. RabbitMQ credentials restored automatically via default-user secret. |
| 60 | DataPlaneNodeSet | OADP Restore CR | DataPlane resources (optional) |

### Restore CRs

Restore CRs are documented in [`restore/README.md`](restore/README.md). Each uses a shared resource modifier ConfigMap that:
- Strips `kubectl.kubernetes.io/last-applied-configuration` annotations
- Adds `deployment-stage: infrastructure-only` annotation to OpenStackControlPlane

Example restore CR for order 20:

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-20-infrastructure
  namespace: openshift-adp
spec:
  backupName: openstack-backup-resources
  includedNamespaces:
  - openstack
  labelSelector:
    matchLabels:
      backup.openstack.org/restore: "true"
      backup.openstack.org/restore-order: "20"
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
```

### RabbitMQ Credentials

RabbitMQ credentials are restored automatically. The infra-operator's RabbitMQ
controller labels the `default-user` secret for restore (order 10). On restore,
the secret is restored before the RabbitMQ cluster is created, and the controller
reuses the existing credentials instead of generating new ones.

### Automated Restore

The ci-framework `cifmw_backup_restore` role orchestrates the full restore flow.
See [`restore/README.md`](restore/README.md) for usage.

## Testing Checklist

### CRD Label Verification

```bash
# List all CRDs with backup-restore labels
oc get crd -l backup.openstack.org/restore=true

# Verify specific CRD labels
oc get crd openstackcontrolplanes.core.openstack.org -o jsonpath='{.metadata.labels}'
```

### Controller Verification

```bash
# Create OpenStackBackupConfig
oc apply -f - <<EOF
apiVersion: backup.openstack.org/v1beta1
kind: OpenStackBackupConfig
metadata:
  name: backup-config
  namespace: openstack
spec: {}
EOF

# Check controller status
oc get openstackbackupconfig backup-config -n openstack

# Verify user-provided secrets are labeled
oc get secret -l backup.openstack.org/restore=true -n openstack

# Verify CR instances are labeled
oc get openstackcontrolplane -l backup.openstack.org/restore=true -n openstack
```

### Backup Testing

```bash
# Verify backup completes
oc get backup -n openshift-adp
oc describe backup openstack-backup-resources -n openshift-adp

# Verify CSI volume snapshots
oc get volumesnapshot -n openstack
```

### Restore Testing

```bash
# Follow the manual restore procedure in restore/README.md
# or use the ci-framework cifmw_backup_restore role
```

## Troubleshooting

### Resources Not Being Labeled

1. Verify OpenStackBackupConfig exists:
   ```bash
   oc get openstackbackupconfig -n openstack
   ```
2. Check controller logs:
   ```bash
   oc logs -n openstack-operators deployment/openstack-operator-controller-manager | grep -i backup
   ```
3. Check if resource has ownerReferences (controller only labels user-provided resources):
   ```bash
   oc get secret <name> -o jsonpath='{.metadata.ownerReferences}'
   ```
4. Check if resource is excluded by label or name filters

### CRD Label Cache Issues

- Verify CRD labels: `oc get crd -l backup.openstack.org/restore=true`
- Cache is built lazily on first reconciliation; restart operator to rebuild
- After `make generate manifests`, apply updated CRDs: `oc apply -f config/crd/bases/`

### PVC Restore Failures

- Verify CSI driver supports snapshots
- Check VolumeSnapshotClass exists
- Verify `restorePVs: true` in Restore CR

### Database Restore Issues

- Verify PVCs were restored first (order 00)
- Check GaleraRestore CR status: `oc get galerarestore -n openstack`
- Check restore pod logs: `oc logs <backup-source>-restore-<restore-name> -n openstack`

## See Also

- Design document: [`backup-restore-controller-design.md`](backup-restore-controller-design.md)
- Backup: [`backup/README.md`](backup/README.md)
- Restore: [`restore/README.md`](restore/README.md)
- ci-framework playbooks: [ci-framework](https://github.com/openstack-k8s-operators/ci-framework)
