# Enhancement Proposal: Backup Resources to PVC

## Summary

Enhance the backup/restore system to serialize OpenStack CRs, Secrets, and ConfigMaps to a dedicated PVC instead of using Velero/OADP object storage for these resources. This would unify the backup mechanism to use CSI volume snapshots exclusively, resulting in a single OADP backup operation instead of two separate backups.

## Current Approach

The POC implementation uses a two-backup approach:

1. **openstack-backup-pvcs**: CSI snapshots of PVCs labeled with `backup.openstack.org/backup: "true"`
   - Glance image storage PVCs
   - MariaDB GaleraBackup PVCs

2. **openstack-backup-resources**: Velero backup of all other resources to object storage
   - OpenStack CRs (ControlPlane, Version, DataPlane, etc.)
   - Secrets and ConfigMaps
   - NetworkAttachmentDefinitions
   - MariaDB CRs (Account, Database, Backup)

### Limitations

- Requires two separate backup operations
- Velero requires S3-compatible object storage backend
- Resources are stored in different locations (volume snapshots vs object storage)
- Two backup schedules to manage
- Additional complexity in backup/restore workflow

## Proposed Enhancement

Replace the Velero object storage backup of resources with a backup controller that serializes resources to a dedicated PVC. The OADP backup would then only need to snapshot PVCs.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Backup Phase                                                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  OpenStackBackup CR (new)                                    │
│         │                                                     │
│         ├──> Backup Controller discovers resources           │
│         │    (using CRD label cache)                         │
│         │                                                     │
│         ├──> Groups resources by restore order               │
│         │    (00, 10, 20, 30, 40, 60)                        │
│         │                                                     │
│         ├──> Serializes to YAML files on PVC                 │
│         │    /backups/<timestamp>/order-00.yaml              │
│         │                          /order-10.yaml              │
│         │                          /order-20.yaml              │
│         │                          /...                        │
│         │                          /metadata.json             │
│         │                                                     │
│         └──> Labels backup PVC with                          │
│              backup.openstack.org/backup: "true"                    │
│                                                               │
│  OADP Backup (single)                                        │
│         └──> CSI snapshots of ALL labeled PVCs               │
│              - Glance PVCs                                   │
│              - GaleraBackup PVCs                             │
│              - Resource backup PVC                           │
│                                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Restore Phase                                                │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  OADP Restore                                                │
│         └──> Restores ALL PVCs from snapshots                │
│              (including resource backup PVC)                 │
│                                                               │
│  OpenStackRestore CR (new)                                   │
│         │                                                     │
│         ├──> Restore Controller mounts backup PVC            │
│         │                                                     │
│         ├──> Reads /backups/<timestamp>/metadata.json        │
│         │                                                     │
│         ├──> Applies resources in order:                     │
│         │    order-00.yaml → PVC references (if needed)      │
│         │    order-10.yaml → Secrets, ConfigMaps, NADs       │
│         │    order-20.yaml → Infrastructure CRs              │
│         │    order-30.yaml → ControlPlane CR                 │
│         │                    (adds deployment-stage)          │
│         │    order-40.yaml → GaleraBackup CRs                │
│         │                                                     │
│         ├──> Waits for user to trigger DB restore            │
│         │    (manual GaleraRestore step)                     │
│         │                                                     │
│         └──> order-60.yaml → DataPlane CRs                   │
│                                                               │
│  During apply:                                               │
│    - Strip metadata.ownerReferences                          │
│    - Strip last-applied-configuration annotation            │
│    - Add deployment-stage annotation to ControlPlane        │
│    - Preserve custom user modifications                     │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## Implementation Details

### New CRDs

#### OpenStackBackup

```go
// OpenStackBackup triggers a backup of OpenStack resources to a PVC
type OpenStackBackup struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   OpenStackBackupSpec   `json:"spec,omitempty"`
    Status OpenStackBackupStatus `json:"status,omitempty"`
}

type OpenStackBackupSpec struct {
    // PVCName is the name of the PVC to store backup data
    // +kubebuilder:default="openstack-backup-resources"
    PVCName string `json:"pvcName,omitempty"`

    // StorageRequest for the backup PVC
    // +kubebuilder:default="10Gi"
    StorageRequest string `json:"storageRequest,omitempty"`

    // StorageClassName for the backup PVC
    // +optional
    StorageClassName *string `json:"storageClassName,omitempty"`

    // ControlPlaneSelector identifies which ControlPlane to backup
    // +optional
    ControlPlaneSelector *metav1.LabelSelector `json:"controlPlaneSelector,omitempty"`
}

type OpenStackBackupStatus struct {
    // Conditions for the backup
    Conditions condition.Conditions `json:"conditions,omitempty"`

    // BackupTimestamp when backup was created
    BackupTimestamp *metav1.Time `json:"backupTimestamp,omitempty"`

    // BackupPath within the PVC where backup is stored
    BackupPath string `json:"backupPath,omitempty"`

    // ResourceCount total resources backed up
    ResourceCount int `json:"resourceCount,omitempty"`

    // BackupSize approximate size of backup data
    BackupSize string `json:"backupSize,omitempty"`
}
```

#### OpenStackRestore

```go
// OpenStackRestore triggers a restore of OpenStack resources from a PVC
type OpenStackRestore struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   OpenStackRestoreSpec   `json:"spec,omitempty"`
    Status OpenStackRestoreStatus `json:"status,omitempty"`
}

type OpenStackRestoreSpec struct {
    // BackupSource references the OpenStackBackup to restore from
    BackupSource string `json:"backupSource"`

    // RestoreOrder controls which restore orders to process
    // If empty, restores all orders up to 40 (before manual DB restore)
    // +optional
    RestoreOrder []int `json:"restoreOrder,omitempty"`

    // WaitForDatabaseRestore pauses before order 60 until DB restore is confirmed
    // +kubebuilder:default=true
    WaitForDatabaseRestore bool `json:"waitForDatabaseRestore,omitempty"`
}

type OpenStackRestoreStatus struct {
    // Conditions for the restore
    Conditions condition.Conditions `json:"conditions,omitempty"`

    // RestoreTimestamp when restore started
    RestoreTimestamp *metav1.Time `json:"restoreTimestamp,omitempty"`

    // CurrentOrder being restored
    CurrentOrder int `json:"currentOrder,omitempty"`

    // ResourcesRestored count of resources applied
    ResourcesRestored int `json:"resourcesRestored,omitempty"`

    // WaitingForDatabaseRestore indicates manual intervention needed
    WaitingForDatabaseRestore bool `json:"waitingForDatabaseRestore,omitempty"`
}
```

### Backup Controller Logic

```go
func (r *OpenStackBackupReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Get OpenStackBackup CR
    backup := &backupv1beta1.OpenStackBackup{}

    // 2. Create or verify backup PVC exists
    pvc := r.ensureBackupPVC(ctx, backup)

    // 3. Create backup job that:
    //    - Mounts the backup PVC
    //    - Runs backup logic in a pod
    job := r.createBackupJob(ctx, backup, pvc)

    // 4. Wait for job completion
    // 5. Update status with backup metadata

    return ctrl.Result{}, nil
}

func backupResources(ctx context.Context, client client.Client, backupPath string) error {
    timestamp := time.Now().Format("20060102-150405")
    backupDir := filepath.Join(backupPath, "backups", timestamp)

    // Create directory structure
    os.MkdirAll(backupDir, 0755)

    // Get CRD label cache to discover resources
    cache := commonbackup.BuildCRDLabelCache(ctx, client)

    // Group resources by restore order
    resourcesByOrder := make(map[int][]unstructured.Unstructured)

    for gvkStr, config := range cache {
        if !config.Enabled {
            continue
        }

        gvk := parseGVK(gvkStr)
        list := &unstructured.UnstructuredList{}
        list.SetGroupVersionKind(gvk)

        // List all resources of this type in namespace
        client.List(ctx, list, client.InNamespace("openstack"))

        for _, item := range list.Items {
            // Clean metadata
            cleanResource(&item)

            // Group by restore order
            order := config.RestoreOrder
            resourcesByOrder[order] = append(resourcesByOrder[order], item)
        }
    }

    // Write each order to separate YAML file
    orders := []int{0, 10, 20, 30, 40, 60}
    for _, order := range orders {
        resources := resourcesByOrder[order]
        if len(resources) == 0 {
            continue
        }

        filename := filepath.Join(backupDir, fmt.Sprintf("order-%02d.yaml", order))
        writeYAMLFile(filename, resources)
    }

    // Write metadata
    metadata := BackupMetadata{
        Timestamp:     timestamp,
        ResourceCount: totalCount,
        Orders:        orders,
    }
    writeJSONFile(filepath.Join(backupDir, "metadata.json"), metadata)

    // Create "latest" symlink
    os.Symlink(timestamp, filepath.Join(backupPath, "backups", "latest"))

    return nil
}

func cleanResource(obj *unstructured.Unstructured) {
    // Remove ownerReferences
    obj.SetOwnerReferences(nil)

    // Remove managed fields
    obj.SetManagedFields(nil)

    // Remove status
    unstructured.RemoveNestedField(obj.Object, "status")

    // Remove runtime metadata
    obj.SetResourceVersion("")
    obj.SetUID("")
    obj.SetGeneration(0)
    obj.SetCreationTimestamp(metav1.Time{})

    // Remove last-applied-configuration
    annotations := obj.GetAnnotations()
    delete(annotations, "kubectl.kubernetes.io/last-applied-configuration")
    obj.SetAnnotations(annotations)
}
```

### Restore Controller Logic

```go
func (r *OpenStackRestoreReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    restore := &backupv1beta1.OpenStackRestore{}

    // 1. Get the backup PVC
    backup := &backupv1beta1.OpenStackBackup{}
    client.Get(ctx, types.NamespacedName{Name: restore.Spec.BackupSource}, backup)

    // 2. Create restore job that:
    //    - Mounts the backup PVC (read-only)
    //    - Runs restore logic in a pod with proper RBAC
    job := r.createRestoreJob(ctx, restore, backup)

    // 3. Monitor job progress via status updates
    // 4. Handle waiting for manual DB restore

    return ctrl.Result{}, nil
}

func restoreResources(ctx context.Context, client client.Client, backupPath string, orders []int) error {
    // Read metadata
    metadata := readMetadata(filepath.Join(backupPath, "backups", "latest", "metadata.json"))

    // Determine which orders to restore
    if len(orders) == 0 {
        orders = []int{0, 10, 20, 30, 40} // Up to DB restore point
    }

    for _, order := range orders {
        filename := filepath.Join(backupPath, "backups", "latest", fmt.Sprintf("order-%02d.yaml", order))

        resources := readYAMLFile(filename)

        for _, resource := range resources {
            // Apply special handling for ControlPlane (order 30)
            if order == 30 && resource.GetKind() == "OpenStackControlPlane" {
                annotations := resource.GetAnnotations()
                if annotations == nil {
                    annotations = make(map[string]string)
                }
                annotations["core.openstack.org/deployment-stage"] = "infrastructure-only"
                resource.SetAnnotations(annotations)
            }

            // Apply resource
            err := client.Create(ctx, &resource)
            if err != nil && !k8s_errors.IsAlreadyExists(err) {
                return err
            }
        }

        // Log progress
        log.Info("Restored resources", "order", order, "count", len(resources))
    }

    return nil
}
```

### OADP Integration

With this enhancement, the OADP backup becomes simpler:

```yaml
---
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: openstack-backup
  namespace: openshift-adp
spec:
  # Include only the openstack namespace
  includedNamespaces:
  - openstack

  # Only backup PVCs (data volumes)
  includedResources:
  - persistentvolumeclaims
  - persistentvolumes

  # Only backup labeled PVCs
  labelSelector:
    matchLabels:
      backup.openstack.org/backup: "true"

  # Use CSI snapshots for all PVCs
  snapshotVolumes: true

  # Storage location for backup metadata
  storageLocation: default

  # TTL for the backup
  ttl: 720h0m0s
```

The restore is equally simple:

```yaml
---
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-pvcs
  namespace: openshift-adp
spec:
  backupName: openstack-backup

  includedNamespaces:
  - openstack

  # Restore PVCs from volume snapshots
  restorePVs: true
```

## Benefits

1. **Unified backup mechanism**: All data backed up via CSI volume snapshots
2. **Simpler workflow**: Single OADP backup/restore operation
3. **Reduced external dependencies**: No S3-compatible object storage needed for resources
4. **Consistent storage location**: Everything in volume snapshots
5. **Better resource tracking**: Backup metadata stored with resources
6. **Version control friendly**: YAML files can be inspected/modified if needed
7. **Preservation of user modifications**: Resources are serialized as-is with customizations
8. **Automated ordering**: Restore controller handles sequencing automatically

## Considerations

### PVC Size Management

- Need to estimate required storage for serialized resources
- Default 10Gi should be sufficient for most deployments
- Allow users to configure via OpenStackBackupSpec.StorageRequest
- Monitor backup size and update status for visibility

### Backup PVC Lifecycle

- PVC should be created by backup controller
- Labeled with `backup.openstack.org/backup: "true"` to be included in OADP backup
- Should persist between backups (accumulate backup history)
- Cleanup policy: keep last N backups, or retention period

### Restore Job RBAC

- Restore job needs broad permissions to create all resource types
- Use dedicated ServiceAccount with appropriate ClusterRole
- Scope permissions to openstack namespace only

### Error Handling

- Partial backup/restore scenarios
- Validation of backup data before restore
- Rollback capability if restore fails mid-process
- Status conditions for observability

### Database Restore Coordination

- Restore controller must pause before order 60
- Set status.WaitingForDatabaseRestore = true
- User creates GaleraRestore CRs manually
- User updates OpenStackRestore to continue (e.g., annotation or spec field)
- Controller removes deployment-stage annotation and proceeds to order 60

### Concurrent Backups

- Lock mechanism to prevent concurrent backups
- Use CR status or ConfigMap lock
- Validate backup not in progress before starting new one

## Migration Path

This enhancement doesn't break the existing POC approach:

1. **Phase 1**: Implement new OpenStackBackup/Restore CRDs alongside current OADP CRs
2. **Phase 2**: Document both approaches, allow users to choose
3. **Phase 3**: Recommend PVC-based approach as default
4. **Phase 4**: Deprecate direct OADP resource backup approach

Users can validate the new approach in test environments before switching production backups.

## Future Enhancements

### Incremental Backups

Instead of full serialization each time, track resource changes:
- Store resource fingerprints (hash of spec)
- Only serialize changed resources
- Maintain index of all resources across backups

### Backup Encryption

- Encrypt YAML files before writing to PVC
- Store encryption key in Secret
- Decrypt during restore

### Cross-Namespace Restore

- Support restoring to different namespace
- Rewrite namespace references in resources
- Handle cross-namespace Secret/ConfigMap references

### Backup Validation

- Verify backup integrity before restore
- Check resource relationships (referenced Secrets exist, etc.)
- Dry-run restore to detect issues

### Automatic Scheduling

- Extend OpenStackBackup with schedule field (cron expression)
- Controller creates backups automatically
- Cleanup old backups based on retention policy

### Backup Compression

- Compress YAML files (gzip, zstd)
- Reduce PVC storage requirements
- Decompress during restore

## Open Questions

1. Should OpenStackBackup be namespaced or cluster-scoped?
   - Recommendation: Namespaced, allows per-ControlPlane backups

2. Should restore be fully automated or require manual approval between stages?
   - Recommendation: Pause at DB restore point, allow manual trigger for continuation

3. How to handle backup PVC when ControlPlane is deleted?
   - Recommendation: Don't set ownerReference, persist independently

4. Should we support backup to multiple PVCs (e.g., one per restore order)?
   - Recommendation: Start simple with single PVC, can enhance later

5. Should backup/restore controllers be separate controllers or extend existing ones?
   - Recommendation: Separate controllers for clear separation of concerns

## Implementation Phases

### Phase 1: Core Backup to PVC (MVP)
- [ ] Define OpenStackBackup CRD
- [ ] Implement backup controller
- [ ] Create backup job that serializes resources to PVC
- [ ] Label backup PVC for OADP inclusion
- [ ] Update documentation

### Phase 2: Restore from PVC
- [ ] Define OpenStackRestore CRD
- [ ] Implement restore controller
- [ ] Create restore job that applies resources from PVC
- [ ] Handle deployment-stage annotation
- [ ] Handle restore ordering

### Phase 3: Database Restore Integration
- [ ] Pause restore before order 60
- [ ] Wait for manual DB restore confirmation
- [ ] Resume restore after confirmation
- [ ] Remove deployment-stage annotation

### Phase 4: Production Readiness
- [ ] Error handling and validation
- [ ] Backup retention/cleanup
- [ ] Monitoring and observability
- [ ] Comprehensive testing
- [ ] User documentation and examples

### Phase 5: Advanced Features
- [ ] Backup scheduling
- [ ] Backup encryption
- [ ] Incremental backups
- [ ] Backup validation

## References

- [POC Implementation Guide](backup-restore-controller-implementation.md)
- [OADP Setup](oadp/README.md)
- [Backup/Restore Design](backup-restore-controller-design.md)
