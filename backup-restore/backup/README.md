# OpenStack Backup CRs

This document describes the OADP Backup CR strategy for backing up OpenStack
environments. The backup playbooks and templates are in the
[ci-framework](https://github.com/openstack-k8s-operators/ci-framework)
`cifmw_backup_restore` role.

## Backup Approach

We use a two-backup strategy:

1. **PVC backup** — PVCs with CSI snapshots
   - Filters at backup time using `labelSelector`
   - Only includes PVCs labeled with `backup.openstack.org/backup: "true"`
   - Supports Data Mover (`snapshotMoveData: true`, default) to upload snapshot
     data to the BackupStorageLocation (S3/MinIO) via Kopia, enabling restore
     even after total cluster loss

2. **Resources backup** — everything except PVCs
   - Backs up all resources in the namespace
   - Excludes PVCs (backed up separately)
   - Includes: CRs, Secrets, ConfigMaps, NADs, etc.

## Why Two Backups?

- **Flexibility**: Allows selective restore (e.g., restore only PVCs or only CRs)
- **PVC Filtering**: Only backup PVCs we explicitly labeled (saves storage costs)
- **Restore Control**: Can restore in stages using `backup-restore-order` labels

## Prerequisites

- OADP operator installed in `openshift-adp` namespace
- Velero storage location configured (named `velero-1`)
- CSI snapshot capability for PVC backups
- OpenStack deployed in `openstack` namespace with backup labels applied
- GaleraBackup CRs configured for each Galera database instance (main cell and
  any additional cells). The CRs define cronjobs that dump databases to PVCs.
  **Note:** When using LVM storage, specify PVC sizes with binary suffixes (e.g.,
  `5Gi` not `5G`) to avoid size rounding issues.

## Quick Start

Use the ci-framework backup playbook (see [top-level README](../README.md)):

```bash
ansible-playbook playbooks/backup_restore.yaml \
  -e cifmw_backup_restore_install_deps=false \
  -e cifmw_backup_restore_run_backup=true \
  -e cifmw_backup_restore_run_cleanup=false \
  -e cifmw_backup_restore_run_restore=false
```

The playbook runs three steps:
1. **Trigger Galera DB dumps** — creates jobs from GaleraBackup cronjobs
2. **OADP PVC backup** — CSI snapshots of PVCs labeled with `backup.openstack.org/backup=true`
3. **OADP resources backup** — all resources in the namespace except PVCs

PVC backup labels are set automatically by service operators (glance-operator, mariadb-operator).

## Manual Backup Procedure

If you are not using the ci-framework playbooks, follow these steps manually.

### Step 1: Collect Operator Version and Backup Timestamp

Record the operator version details and generate a backup timestamp. The
operator version is stored as annotations on the Backup CRs so you can
identify which operator version to install on the target cluster before
restoring.

```bash
BACKUP_TS=$(date +%Y%m%d-%H%M%S)

CSV_VERSION=$(oc get csv -n openstack-operators \
  -l operators.coreos.com/openstack-operator.openstack-operators \
  -o jsonpath='{.items[0].metadata.name}')
CATALOG_IMG=$(oc get catalogsource -n openstack-operators \
  -o jsonpath='{.items[0].spec.image}')
OPERATOR_IMG=$(oc get deployment openstack-operator-controller-manager \
  -n openstack-operators -o jsonpath='{.spec.template.spec.containers[0].image}')

echo "Backup timestamp: ${BACKUP_TS}"
echo "CSV version:      ${CSV_VERSION}"
echo "Catalog image:    ${CATALOG_IMG}"
echo "Operator image:   ${OPERATOR_IMG}"
```

### Step 2: Trigger Galera Database Dumps

Create backup jobs from the GaleraBackup cronjobs, passing the backup timestamp
so DB dump filenames match the OADP backup name:

```bash

for CRONJOB in $(oc get cronjob -n openstack -l app=galera -o jsonpath='{.items[*].metadata.name}'); do
  JOB_NAME="${CRONJOB}-${BACKUP_TS}"
  oc -n openstack create job --from=cronjob/${CRONJOB} ${JOB_NAME} \
    --dry-run=client -o json | \
    jq '.spec.template.spec.containers[0].env += [{"name":"BACKUP_TIMESTAMP","value":"'${BACKUP_TS}'"}]' | \
    oc -n openstack create -f -
done

# Wait for all backup jobs to complete
oc -n openstack wait --for=condition=complete job -l app=galera --timeout=10m
```

### Step 3: OADP PVC Backup (CSI Snapshots)

Create an OADP Backup CR for PVCs labeled with `backup.openstack.org/backup=true`:

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: openstack-backup-pvcs-${BACKUP_TS}
  namespace: openshift-adp
  annotations:
    openstack.org/csv-version: "${CSV_VERSION}"
    openstack.org/catalog-source-image: "${CATALOG_IMG}"
    openstack.org/operator-image: "${OPERATOR_IMG}"
spec:
  includedNamespaces:
  - openstack
  labelSelector:
    matchLabels:
      backup.openstack.org/backup: "true"
  snapshotVolumes: true
  defaultVolumesToFsBackup: false
  snapshotMoveData: true
  volumeSnapshotLocations: []
  storageLocation: velero-1
  ttl: 720h
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  backup/openstack-backup-pvcs-${BACKUP_TS} -n openshift-adp --timeout=30m
```

Remove `snapshotMoveData: true` if you don't want to upload snapshots to
the BackupStorageLocation (local CSI snapshots only).

### Step 4: OADP Resources Backup

Create an OADP Backup CR for all resources except PVCs:

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: openstack-backup-resources-${BACKUP_TS}
  namespace: openshift-adp
  annotations:
    openstack.org/csv-version: "${CSV_VERSION}"
    openstack.org/catalog-source-image: "${CATALOG_IMG}"
    openstack.org/operator-image: "${OPERATOR_IMG}"
spec:
  includedNamespaces:
  - openstack
  excludedResources:
  - persistentvolumeclaims
  - persistentvolumes
  storageLocation: velero-1
  ttl: 720h
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  backup/openstack-backup-resources-${BACKUP_TS} -n openshift-adp --timeout=30m
```

### Step 5: Verify

```bash
oc get backup -n openshift-adp -o custom-columns=NAME:.metadata.name,PHASE:.status.phase

# Verify operator version annotations are recorded
oc get backup -n openshift-adp -o custom-columns=\
NAME:.metadata.name,\
CSV:.metadata.annotations.openstack\.org/csv-version,\
CATALOG:.metadata.annotations.openstack\.org/catalog-source-image
```

## Backup Contents

### backup-openstack-pvcs

All PVCs labeled with `backup.openstack.org/backup: "true"` are backed up via
CSI volume snapshots. Service operators set this label automatically on PVCs
they manage (e.g., Glance image storage, GaleraBackup database dumps).

### backup-openstack-resources

All resources in the namespace except PVCs. This includes CRs, Secrets,
ConfigMaps, NetworkAttachmentDefinitions, and any other namespace-scoped
resources. Restore ordering is controlled by `backup.openstack.org/restore-order`
labels on the backed-up resources (see [`../restore/README.md`](../restore/README.md)).

## Verification

```bash
# List all backups
oc get backup -n openshift-adp

# Check backup details and status
oc get backup -n openshift-adp -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,ERRORS:.status.errors,WARNINGS:.status.warnings

# List volume snapshots created by backup
oc get volumesnapshot -n openstack
oc get volumesnapshotcontent
```

### Data Mover Verification

When using `snapshotMoveData: true`, verify that all PVC data was uploaded
to the BackupStorageLocation:

```bash
# Check that DataUpload CRs completed for each PVC
oc get datauploads -n openshift-adp -l velero.io/backup-name=openstack-backup-pvcs
oc get datauploads -n openshift-adp -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,BYTES:.status.progress.totalBytes

# All DataUploads should show Phase=Completed
# If any show Phase=Failed, check node-agent logs:
oc logs -n openshift-adp -l name=node-agent --tail=50
```

## See Also

- Restore: [`../restore/README.md`](../restore/README.md)
- Implementation guide: [`../backup-restore-controller-implementation.md`](../backup-restore-controller-implementation.md)
- Design document: [`../backup-restore-controller-design.md`](../backup-restore-controller-design.md)
