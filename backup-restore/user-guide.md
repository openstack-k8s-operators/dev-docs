# OpenStack Backup and Restore User Guide

This guide describes how to back up and restore an OpenStack control plane
deployed on OpenShift using OADP (OpenShift API for Data Protection).

## How It Works

### Full Backup, Selective Restore

The backup captures a complete snapshot of the OpenStack namespace. The restore
is selective — only resources that operators cannot recreate are restored:

| Restored | Not restored (operators rebuild) |
|---|---|
| User-provided Secrets (passwords, SSH keys) | Operator-managed Secrets |
| User-provided ConfigMaps | Operator-managed ConfigMaps |
| CA certificates | TLS service certificates (reissued) |
| PVCs with data (Glance images, DB dumps) | Service deployments |
| User-created CRs (ControlPlane, Version, NodeSet) | Service CRs (Nova, Keystone, Galera, ...) |
| RabbitMQ default-user credentials | MariaDBAccount/MariaDBDatabase CRs |

The restored resources serve as "seeds" — operators reconcile and recreate
everything else from those inputs.

### Labeling

Resources are selected for restore using Kubernetes labels
(`backup.openstack.org/restore`, `backup.openstack.org/restore-order`).
Labels are applied automatically:

- **OpenStackBackupConfig controller** labels CR instances and user-provided
  Secrets, ConfigMaps, and NADs
- **Operators** label PVCs, CA cert secrets, custom Issuers, and RabbitMQ
  credentials at creation time

No manual labeling is required for standard deployments. PVCs passed via
`extraMounts` must be labeled manually (see [ExtraMounts PVCs](#extramounts-pvcs)).

### Staged Deployment

The restore uses staged deployment to ensure databases are restored before
services start. The OpenStackControlPlane is restored with a
`deployment-stage: infrastructure-only` annotation, which starts only
infrastructure services (Galera, OVN, RabbitMQ, Memcached). After databases
are restored into running Galera, the annotation is removed and all services
start with the restored data.

### Backup Storage

OADP uploads all backup data to S3-compatible storage (MinIO, ODF, AWS S3,
etc.) via the Data Mover. This enables restore even on a completely new
cluster — Velero discovers existing backups from the object storage.

## Prerequisites

1. **OpenShift 4.20+** with **OADP 1.5+** (ships Velero v1.16+ with Data Mover improvements)
2. **OADP operator** installed in `openshift-adp` namespace
3. **BackupStorageLocation** configured and healthy
4. **VolumeSnapshotClass** with Velero label (`velero.io/csi-volumesnapshot-class=true`)
5. **GaleraBackup CRs** configured for each Galera database instance
6. **OpenStack deployed** and healthy in the `openstack` namespace

### Verify Prerequisites

```bash
# OADP operator
oc get crd backups.velero.io

# BackupStorageLocation
oc get backupstoragelocation -n openshift-adp

# VolumeSnapshotClass with Velero label
oc get volumesnapshotclass -l velero.io/csi-volumesnapshot-class=true

# OpenStackBackupConfig (auto-created by the operator)
oc get openstackbackupconfig -n openstack

# Labeled resources
echo "Secrets:    $(oc get secret -n openstack -l backup.openstack.org/restore=true --no-headers | wc -l)"
echo "ConfigMaps: $(oc get configmap -n openstack -l backup.openstack.org/restore=true --no-headers | wc -l)"
echo "PVCs:       $(oc get pvc -n openstack -l backup.openstack.org/backup=true --no-headers | wc -l)"
```

### Storage Requirements

- PVC sizes **must use binary units** (`5Gi`, not `5G`) when using LVM storage
- The target cluster must support the same StorageClasses as the source

### Velero Version

```bash
VELERO_POD=$(oc get pods -n openshift-adp -l deploy=velero \
  --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')
oc exec -n openshift-adp ${VELERO_POD} -- /velero version
```

## Backup Procedure

### Pre-check: Verify Control Plane Health

Verify that the OpenStack control plane and data plane are Ready before
taking a backup. A backup of an unhealthy environment might restore into
the same broken state.

```bash
oc wait openstackcontrolplane -n openstack --all \
  --for=condition=Ready --timeout=10s
oc wait openstackdataplanenodeset -n openstack --all \
  --for=condition=Ready --timeout=10s
```

### Step 1: Collect Operator Version and Backup Timestamp

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

Create backup jobs from the GaleraBackup cronjobs:

```bash
for CRONJOB in $(oc get cronjob -n openstack -l app=galera \
  -o jsonpath='{.items[*].metadata.name}'); do
  JOB_NAME="${CRONJOB}-${BACKUP_TS}"
  oc -n openstack create job --from=cronjob/${CRONJOB} ${JOB_NAME} \
    --dry-run=client -o json | \
    jq '.spec.template.spec.containers[0].env += [{"name":"BACKUP_TIMESTAMP","value":"'${BACKUP_TS}'"}]' | \
    oc -n openstack create -f -
done

# Wait for all backup jobs to complete
oc -n openstack wait --for=condition=complete job -l app=galera --timeout=10m
```

### Step 3: OVN Database Backup (optional)

Back up the OVN Northbound and Southbound databases using `ovsdb-client backup`.
The backup files are written to the OVN PVCs, which are captured by the OADP PVC
backup in the next step. If you skip this step, see
Step 10 (Sync Neutron to OVN).

OVN DB PVCs are not labeled for backup by default. Label them first:

```bash
oc label pvc -n openstack -l service=ovsdbserver-nb \
  backup.openstack.org/backup=true \
  backup.openstack.org/restore=true \
  backup.openstack.org/restore-order=00 \
  backup.openstack.org/category=controlplane
oc label pvc -n openstack -l service=ovsdbserver-sb \
  backup.openstack.org/backup=true \
  backup.openstack.org/restore=true \
  backup.openstack.org/restore-order=00 \
  backup.openstack.org/category=controlplane
```

Then trigger the OVN database backups:

```bash
oc exec ovsdbserver-nb-0 -n openstack -- bash -c \
  "ovsdb-client backup unix:/etc/ovn/ovnnb_db.sock OVN_Northbound \
    > /etc/ovn/ovnnb_db.db.${BACKUP_TS}"

oc exec ovsdbserver-sb-0 -n openstack -- bash -c \
  "ovsdb-client backup unix:/etc/ovn/ovnsb_db.sock OVN_Southbound \
    > /etc/ovn/ovnsb_db.db.${BACKUP_TS}"
```

### Step 4: OADP PVC Backup

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
      backup.openstack.org/backup: "true"  # Only PVCs labeled by operators
  snapshotVolumes: true                    # Use CSI volume snapshots
  defaultVolumesToFsBackup: false          # Do not use file system backup
  snapshotMoveData: true                   # Upload snapshots to S3 via Data Mover
  volumeSnapshotLocations: []              # Use default CSI driver, not a specific VSL
  storageLocation: velero-1                # BackupStorageLocation name
  ttl: 720h                                # Backup retention (30 days)
  hooks:                                   # Swift xattr backup (optional, see notes)
    resources:
      - name: swift-xattr-backup
        labelSelector:
          matchLabels:
            component: swift-storage
        pre:
          - exec:
              container: object-server
              command:
                - /bin/bash
                - -c
                - |
                  set -e
                  DUMP="/srv/node/pv/.swift-xattrs.dump"
                  rm -f "\$DUMP" "\${DUMP}.applied" "\${DUMP}.missing"
                  getfattr -R -d -m user.swift /srv/node/pv/ 1> "\$DUMP"
                  echo "xattr backup complete: \$(grep -c '^# file:' "\$DUMP") files"
              onError: Fail
              timeout: 600s
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  backup/openstack-backup-pvcs-${BACKUP_TS} -n openshift-adp --timeout=30m
```

**Notes:**
- The `hooks.resources` section is required when Swift is deployed as part of
  the control plane. It dumps Swift extended attributes (xattrs) to a file on
  the PVC before the CSI snapshot. OADP DataMover uses kopia for
  filesystem-level data transfer, which does not preserve xattrs. Swift stores
  all object metadata in xattrs — without this hook, all Swift objects become
  inaccessible after restore. The hook is safe to include even if Swift is not
  deployed — it only targets pods with `component: swift-storage` and is
  silently skipped if no matching pods exist. On the restore side, an init
  container on the Swift storage StatefulSet restores xattrs from the dump
  before Swift services start. See the
  [swift-operator backup/restore documentation](https://github.com/openstack-k8s-operators/swift-operator/blob/main/docs/backup-restore-xattrs.md)
  for details.
- Remove `snapshotMoveData: true` if you do not want to upload snapshot data
  to the BackupStorageLocation (local CSI snapshots only). Without Data Mover,
  PVC restore requires the original VolumeSnapshotContent to still exist.
- Depending on the storage backend, you may need to configure
  `volumeSnapshotLocations` or adjust CSI driver settings. See the
  [OADP documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/backup_and_restore/oadp-application-backup-and-restore)
  for storage-specific configuration.

### Step 5: OADP Resources Backup

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
  excludedResources:              # PVCs are backed up separately in Step 3
  - persistentvolumeclaims
  - persistentvolumes
  storageLocation: velero-1       # BackupStorageLocation name
  ttl: 720h                       # Backup retention (30 days)
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  backup/openstack-backup-resources-${BACKUP_TS} -n openshift-adp --timeout=30m
```

### Step 6: OADP Baremetal Backup (optional — baremetal-provisioned nodes only)

Skip this step if all OpenStackDataPlaneNodeSets use `preProvisioned: true`.

If your Metal3 BaremetalHosts are in the same namespace as the OpenStack
resources, they are already captured by the resources backup in Step 5 and
this step can be skipped.

If your BaremetalHosts are in a different namespace, create an additional
backup for that namespace. During restore, the secrets and configmaps from
this backup — BaremetalHosts, secrets, and configmaps — are restored in
Restore Step 12:

```bash
BMH_NAMESPACE=<namespace where BaremetalHosts are located>
BMH_BACKUP=openstack-backup-bmh-${BACKUP_TS}

cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: ${BMH_BACKUP}
  namespace: openshift-adp
  annotations:
    openstack.org/csv-version: "${CSV_VERSION}"
    openstack.org/catalog-source-image: "${CATALOG_IMG}"
    openstack.org/operator-image: "${OPERATOR_IMG}"
spec:
  includedNamespaces:
  - ${BMH_NAMESPACE}
  includedResources:
  - baremetalhosts.metal3.io
  - secrets
  - configmaps
  storageLocation: velero-1
  ttl: 720h
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  backup/${BMH_BACKUP} -n openshift-adp --timeout=30m
```

The `OpenStackBaremetalSet` and `OpenStackProvisionServer` resources are in
the `openstack` namespace and are already captured by the resources backup
in Step 5.

### Step 7: Verify Backup

```bash
# Check backup status
oc get backup -n openshift-adp -o custom-columns=NAME:.metadata.name,PHASE:.status.phase

# Verify operator version annotations
oc get backup -n openshift-adp -o custom-columns=\
NAME:.metadata.name,\
CSV:.metadata.annotations.openstack\.org/csv-version,\
CATALOG:.metadata.annotations.openstack\.org/catalog-source-image

# Check Data Mover uploads completed (PVC backup)
oc get datauploads -n openshift-adp \
  -l velero.io/backup-name=openstack-backup-pvcs-${BACKUP_TS} \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase
```

## Restore Procedure

**Note:** The `--timeout` values in the steps below are defaults for typical
deployments. Large PVC sizes or slow storage backends may require longer
timeouts, especially for the PVC restore (Step 1) and Data Mover downloads.
Adjust as needed for your environment.

### Pre-check: Verify Operator Version

The target cluster must have the **same operator version** as the source.

```bash
BACKUP_TIMESTAMP=<your-backup-timestamp>
RESOURCES_BACKUP=openstack-backup-resources-${BACKUP_TIMESTAMP}
PVC_BACKUP=openstack-backup-pvcs-${BACKUP_TIMESTAMP}
RESTORE_SUFFIX=$(date +%Y%m%d-%H%M%S)

# Required version (from backup)
echo "=== Required operator version ==="
oc get backup ${RESOURCES_BACKUP} -n openshift-adp -o custom-columns=\
CSV:.metadata.annotations.openstack\.org/csv-version,\
CATALOG:.metadata.annotations.openstack\.org/catalog-source-image,\
OPERATOR:.metadata.annotations.openstack\.org/operator-image

# Installed version
echo "=== Installed operator version ==="
oc get csv -n openstack-operators \
  -l operators.coreos.com/openstack-operator.openstack-operators \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase
```

If versions do not match, install the correct operator version before proceeding.

### Pre-check: Clean Namespace

Ensure the target namespace is clean — no stale PVCs or resources from a
previous deployment. Follow the official cleanup procedure:
[Removing RHOSO resources from the RHOCP environment](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/maintaining_the_red_hat_openstack_services_on_openshift_deployment/assembly_removing-rhoso-deployment-from-rhocp-environment#proc_removing-RHOSO-resources-from-RHOCP-environment_clean)

### Step 0: Create OADP Resource Modifier ConfigMap

Create a Velero Resource Modifier ConfigMap in the OADP namespace
(`openshift-adp`). This is an OADP/Velero feature that modifies resources
during restore. The ConfigMap is referenced by all Restore CRs via
`spec.resourceModifier`. It strips ownerReferences and annotations, adds
staged deployment to the ControlPlane, disables InstanceHa during restore,
and pauses BaremetalHosts to prevent Metal3 from acting on them before their
status is restored.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: openstack-restore-resource-modifiers
  namespace: openshift-adp
data:
  resource-modifiers.yaml: |
    version: v1
    resourceModifierRules:
    # Rule 1: Apply to all resources — strip stale ownerReferences (UIDs
    # from the original cluster) to prevent garbage collection, and remove
    # last-applied-configuration annotation (can be too large for etcd)
    - conditions:
        groupResource: "*"
      mergePatches:
      - patchData: |
          metadata:
            ownerReferences: null
            annotations:
              kubectl.kubernetes.io/last-applied-configuration: null
    # Rule 2: OpenStackControlPlane — add staged deployment annotation so
    # only infrastructure services (Galera, OVN, RabbitMQ, Memcached) start.
    # Databases are restored before removing the annotation in Step 9.
    - conditions:
        groupResource: openstackcontrolplanes.core.openstack.org
      mergePatches:
      - patchData: |
          metadata:
            annotations:
              kubectl.kubernetes.io/last-applied-configuration: null
              core.openstack.org/deployment-stage: "infrastructure-only"
    # Rule 3: InstanceHa — disable fencing during restore to prevent
    # false positives while infrastructure is coming up.
    - conditions:
        groupResource: instancehas.instanceha.openstack.org
      mergePatches:
      - patchData: |
          spec:
            disabled: "True"
    # Rule 4: BaremetalHosts — pause BMHs during restore so Metal3 does
    # not act on them before their status is restored.
    - conditions:
        groupResource: baremetalhosts.metal3.io
      mergePatches:
      - patchData: |
          metadata:
            annotations:
              baremetalhost.metal3.io/paused: ""
    # Rule 5: OpenStackBaremetalSets — skip webhook validation and pause
    # reconciliation during restore. Velero restores spec and status in
    # separate API calls; the webhook rejects the restore because the
    # status is initially empty, and the reconciler would act on
    # incomplete state before the status is set.
    - conditions:
        groupResource: openstackbaremetalsets.baremetal.openstack.org
      mergePatches:
      - patchData: |
          metadata:
            annotations:
              openstack.org/skip-webhook-validation: ""
              openstack.org/paused: ""
EOF
```

### Step 1: Restore PVCs

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-00-pvcs-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${PVC_BACKUP}                          # PVC backup from Step 4
  includedNamespaces:
  - openstack
  excludedResources:
  - pods                                             # Exclude pods (Velero includes them as
                                                     # related items, they fail without ServiceAccounts)
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers       # Strips ownerRefs, annotations (Step 0)
  restorePVs: true                                   # Restore PV data from snapshots/Data Mover
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-00-pvcs-${RESTORE_SUFFIX} -n openshift-adp --timeout=15m
```

### Step 2: Restore Secrets, ConfigMaps, NADs

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-10-foundation-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${RESOURCES_BACKUP}                    # Resources backup from Step 5
  includedNamespaces:
  - openstack
  labelSelector:                                     # Only restore resources labeled for
    matchLabels:                                     # this specific restore order
      backup.openstack.org/restore: "true"
      backup.openstack.org/restore-order: "10"
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-10-foundation-${RESTORE_SUFFIX} -n openshift-adp --timeout=5m
```

Steps 3-6 follow the same pattern — only the restore order label
and the Restore CR name change. Each step restores a specific set of
resources selected by `backup.openstack.org/restore-order`.

### Step 3: Restore Infrastructure CRs

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-20-infra-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${RESOURCES_BACKUP}
  includedNamespaces:
  - openstack
  labelSelector:
    matchLabels:
      backup.openstack.org/restore: "true"
      backup.openstack.org/restore-order: "20"
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-20-infra-${RESTORE_SUFFIX} -n openshift-adp --timeout=5m
```

### Step 4: Restore OpenStackControlPlane

The ControlPlane is restored with `deployment-stage: infrastructure-only`,
starting only Galera, OVN, RabbitMQ, and Memcached.

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-30-ctlplane-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${RESOURCES_BACKUP}
  includedNamespaces:
  - openstack
  labelSelector:
    matchLabels:
      backup.openstack.org/restore: "true"
      backup.openstack.org/restore-order: "30"
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-30-ctlplane-${RESTORE_SUFFIX} -n openshift-adp --timeout=5m
```

### Step 5: Wait for Infrastructure Ready

Wait for the ControlPlane to become ready in infrastructure-only mode
(Galera, OVN, RabbitMQ, and Memcached):

```bash
oc wait openstackcontrolplane -n openstack --all \
  --for=condition=OpenStackControlPlaneInfrastructureReady --timeout=15m
```

### Step 6: Restore GaleraBackup, IPSet, DataPlaneService

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-40-backup-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${RESOURCES_BACKUP}
  includedNamespaces:
  - openstack
  labelSelector:
    matchLabels:
      backup.openstack.org/restore: "true"
      backup.openstack.org/restore-order: "40"
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-40-backup-${RESTORE_SUFFIX} -n openshift-adp --timeout=5m
```

### Step 7: Service Database Restore

Create GaleraRestore CRs for each Galera instance:

```bash
# List GaleraBackup CRs
oc get galerabackup -n openstack

# Create GaleraRestore for main instance
cat <<EOF | oc apply -f -
apiVersion: mariadb.openstack.org/v1beta1
kind: GaleraRestore
metadata:
  name: openstack
  namespace: openstack
spec:
  backupSource: openstack
EOF

# Create GaleraRestore for additional cells (if multi-cell)
# Repeat for each cell, e.g.:
# cat <<EOF | oc apply -f -
# apiVersion: mariadb.openstack.org/v1beta1
# kind: GaleraRestore
# metadata:
#   name: openstack-cell1
#   namespace: openstack
# spec:
#   backupSource: openstack-cell1
# EOF
```

Wait for restore pods to be ready. You can identify all restore pods using
the `galerarestore/name` label:

```bash
oc get pod -l galerarestore/name -n openstack
NAME                      READY   STATUS    RESTARTS   AGE
restore-openstack         1/1     Running   0          5m
restore-openstack-cell1   1/1     Running   0          5m
```

Execute the restore on each pod (use `--content data` to restore only data,
not grants):

```bash
oc exec -n openstack restore-openstack -- \
  /var/lib/backup-scripts/restore_galera --yes --content data \
  "/backup/data/*_${BACKUP_TS}.sql.gz"

# Repeat for additional cells, e.g.:
# oc exec -n openstack restore-openstack-cell1 -- \
#   /var/lib/backup-scripts/restore_galera --yes --content data \
#   "/backup/data/*_${BACKUP_TS}.sql.gz"
```

Clean up GaleraRestore CRs:

```bash
oc delete galerarestore --all -n openstack
```

### Step 8: OVN Database Restore (optional — only if Backup Step 3 was performed)

Restore the OVN NB and SB databases from the files taken in Backup Step 3.
The backup files are already on the restored PVCs from Step 1. Replace the
database on replica-0 with the backup, delete the database files on replicas
1 and 2 (they contain stale raft membership from the original cluster), then
force delete all OVN pods to restart with the restored data. On restart,
replica-0 bootstraps a new raft cluster from the standalone backup and
replicas 1+2 join automatically.

```bash
REPLICAS=3

# Replace the active DB with the backup on replica-0, delete on replicas 1+
for db in nb sb; do
  oc exec ovsdbserver-${db}-0 -n openstack -c ovsdbserver-${db} -- bash -c \
    "rm /etc/ovn/ovn${db}_db.db && \
     cp /etc/ovn/ovn${db}_db.db.${BACKUP_TS} /etc/ovn/ovn${db}_db.db"

  for ((i=1; i<${REPLICAS}; i++)); do
    oc exec ovsdbserver-${db}-${i} -n openstack -c ovsdbserver-${db} -- \
      rm -f /etc/ovn/ovn${db}_db.db
  done
done

# Force delete all OVN DB pods to restart with the restored database
oc delete pod -n openstack -l service=ovsdbserver-nb --force --grace-period=0
oc delete pod -n openstack -l service=ovsdbserver-sb --force --grace-period=0
oc wait pod -n openstack -l service=ovsdbserver-nb --for=condition=Ready --timeout=5m
oc wait pod -n openstack -l service=ovsdbserver-sb --for=condition=Ready --timeout=5m

# Restart OVN control plane pods to reconnect to the restored databases
oc delete pod -n openstack -l service=ovn-northd
oc delete pod -n openstack -l service=ovn-controller
oc delete pod -n openstack -l service=ovn-controller-ovs
oc delete pod -n openstack -l service=ovn-controller-metrics
```

### Step 9: Resume Full Deployment

Remove the `deployment-stage` annotation to start all OpenStack services:

```bash
oc patch openstackcontrolplane \
  $(oc get openstackcontrolplane -n openstack -o jsonpath='{.items[0].metadata.name}') \
  -n openstack --type json \
  -p '[{"op": "remove", "path": "/metadata/annotations/core.openstack.org~1deployment-stage"}]'
```

Wait for the control plane to be ready:

```bash
oc wait openstackcontrolplane -n openstack --all \
  --for=condition=Ready --timeout=30m
```

### Step 10: Verify and Sync Neutron to OVN

If OVN database backups were not taken (Backup Steps 3 and 8 skipped), the OVN
databases are empty after restore. Run `neutron-ovn-db-sync-util` in `repair`
mode to repopulate the OVN NB/SB databases from Neutron's MariaDB before
proceeding with the EDPM deployment (Step 15). This prevents data plane
outage — without this step, `ovn-controller` would reconnect to empty databases
during the EDPM deployment, wiping cached OVS datapath flows and breaking VM
network connectivity.

Even if OVN DB backups were restored (Step 8), run the sync in `log` mode
first to check for any drift between the Neutron and OVN databases:

```bash
# Check for inconsistencies (log mode — read-only)
oc exec -n openstack -c neutron-api deploy/neutron -- \
  neutron-ovn-db-sync-util \
  --config-file /usr/share/neutron/neutron-dist.conf \
  --config-file /etc/neutron/neutron.conf \
  --config-dir /etc/neutron/neutron.conf.d \
  --ovn-neutron_sync_mode=log --debug
```

If inconsistencies are found (or if Backup Step 3 was skipped), run in `repair`
mode:

```bash
# Fix inconsistencies (repair mode)
oc exec -n openstack -c neutron-api deploy/neutron -- \
  neutron-ovn-db-sync-util \
  --config-file /usr/share/neutron/neutron-dist.conf \
  --config-file /etc/neutron/neutron.conf \
  --config-dir /etc/neutron/neutron.conf.d \
  --ovn-neutron_sync_mode=repair --debug
```

### Step 11: Sync Octavia to OVN (optional — only when Octavia is deployed)

If Octavia with the OVN provider is deployed, the OVN NB database also needs to
be repopulated with Octavia load balancer entries. Similar to the Neutron sync
(Step 10), `octavia-ovn-db-sync-util` recreates any missing OVN load balancer
entries from Octavia's database.

```bash
# Sync Octavia load balancers to OVN NB database
oc exec -n openstack -c octavia-api deploy/octavia-api -- octavia-ovn-db-sync-util
```

The tool automatically discovers all OVN-provider load balancers, listeners,
pools, and members from Octavia's database and recreates them in the OVN NB
database.

Reference:
- [Upstream docs](https://docs.openstack.org/ovn-octavia-provider/latest/admin/driver.html#octavia-db-to-ovn-database-population)
- [RHOSO docs](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/configuring_load_balancing_as_a_service/troubleshoot-maintain-lb-service_rhoso-lbaas#about_synch-lbs-ovn-provider_trbls-lbs)

> **Note:** In future versions, this tool will be integrated into
> `neutron-ovn-db-sync-util` as a
> [plugin](https://docs.openstack.org/ovn-octavia-provider/latest/admin/ovn-db-sync-plugin.html#synchronizing-only-octavia-resources),
> removing the need for a separate step.

### Step 12: Restore BaremetalHosts (optional — baremetal-provisioned nodes only)

Skip this step if all OpenStackDataPlaneNodeSets use `preProvisioned: true`.

Restore Metal3 BaremetalHosts with their status preserved. The resource
modifier from Step 0 injects a `baremetalhost.metal3.io/paused` annotation
so that Metal3 does not act on the BMHs before their status is set.

**Namespace considerations:**

- If your BaremetalHosts are in the `openstack` namespace, their secrets
  and configmaps are already restored in Step 2 (order 10). Skip the
  secrets/configmaps restore below and only restore the BaremetalHosts.
  Use `${RESOURCES_BACKUP}` as the `backupName` and set
  `includedNamespaces` to `openstack`.
- If your BaremetalHosts are in a **separate namespace**, restore their
  secrets and configmaps first from the BMH backup (Backup Step 6),
  then restore the BaremetalHosts themselves.

**Separate namespace only** — restore BMH secrets and configmaps:

```bash
BMH_NAMESPACE=<namespace where BaremetalHosts are located>
BMH_BACKUP=openstack-backup-bmh-${BACKUP_TIMESTAMP}

cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-bmh-secrets-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${BMH_BACKUP}
  includedNamespaces:
  - ${BMH_NAMESPACE}
  includedResources:
  - secrets
  - configmaps
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-bmh-secrets-${RESTORE_SUFFIX} -n openshift-adp --timeout=5m
```

Restore the BaremetalHosts:

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-bmh-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${BMH_BACKUP}
  includedNamespaces:
  - ${BMH_NAMESPACE}
  includedResources:
  - baremetalhosts.metal3.io
  restorePVs: false
  restoreStatus:
    includedResources:
    - baremetalhosts.metal3.io
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-bmh-${RESTORE_SUFFIX} -n openshift-adp --timeout=10m
```

After the restore completes, remove the pause annotations so Metal3
recognizes the BMHs in their restored state. Consider unpausing a
single BMH first to verify it reaches the expected state before
unpausing the rest:

```bash
oc annotate bmh -n ${BMH_NAMESPACE} <single-bmh-name> baremetalhost.metal3.io/paused-
oc get bmh -n ${BMH_NAMESPACE} <single-bmh-name>

# After confirming the BMH is in the expected state, unpause the rest
oc annotate bmh -n ${BMH_NAMESPACE} --all baremetalhost.metal3.io/paused-
```

Verify BMHs are in the expected state:

```bash
oc get bmh -n ${BMH_NAMESPACE}
```

All BMHs should show `STATE: provisioned` and `ONLINE: true`.

### Step 13: Restore OpenStackBaremetalSets (optional — baremetal-provisioned nodes only)

Skip this step if all OpenStackDataPlaneNodeSets use `preProvisioned: true`.

The resource modifier from Step 0 injects two annotations on
`OpenStackBaremetalSet` resources during restore:

- `openstack.org/skip-webhook-validation` — bypasses webhook validation.
  Velero restores spec and status in separate API calls; the webhook
  rejects the restore because the status is initially empty.
- `openstack.org/paused` — pauses operator reconciliation. Without this,
  the operator would act on the incomplete resource state before Velero
  restores the status.

Secrets and configmaps are **not** restored here — they are already
restored in order 10 (Step 2).

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-55-bmset-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${RESOURCES_BACKUP}
  includedNamespaces:
  - openstack
  includedResources:
  - openstackbaremetalsets.baremetal.openstack.org
  - openstackprovisionservers.baremetal.openstack.org
  restoreStatus:
    includedResources:
    - openstackbaremetalsets.baremetal.openstack.org
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-55-bmset-${RESTORE_SUFFIX} -n openshift-adp --timeout=10m
```

Remove the annotations to allow webhook validation and operator
reconciliation to resume:

```bash
oc annotate openstackbaremetalset --all -n openstack \
  openstack.org/skip-webhook-validation- \
  openstack.org/paused-
```

The `OpenStackBaremetalSet` will temporarily leave the `Ready` state while
the `OpenStackProvisionServer` reconciles, but the associated BMHs remain
untouched and their workloads are unaffected.

### Step 14: Restore DataPlane

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-60-dataplane-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${RESOURCES_BACKUP}
  includedNamespaces:
  - openstack
  labelSelector:
    matchLabels:
      backup.openstack.org/restore: "true"
      backup.openstack.org/restore-order: "60"
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-60-dataplane-${RESTORE_SUFFIX} -n openshift-adp --timeout=5m
```

### Step 15: EDPM Deployment

Resync deployment/configuration state from restored backup on dataplane nodes:

```bash
NODESETS=$(oc get openstackdataplanenodeset -n openstack \
  -o jsonpath='{.items[*].metadata.name}')
if [ -n "$NODESETS" ]; then
  cat <<EOF | oc apply -f -
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneDeployment
metadata:
  name: edpm-deployment-post-restore
  namespace: openstack
spec:
  nodeSets:
$(for ns in ${NODESETS}; do echo "  - ${ns}"; done)
EOF
fi
```

### Step 16: Re-enable InstanceHa (optional)

Only required if InstanceHa was used in the backed-up environment.
After verifying the restored cloud is fully operational (see [Verification](#verification)):

```bash
oc patch instanceha \
  $(oc get instanceha -n openstack -o jsonpath='{.items[0].metadata.name}') \
  -n openstack --type merge -p '{"spec":{"disabled":"False"}}'
```

## Verification

```bash
# Control plane status
oc get openstackcontrolplane -n openstack

# All pods running
oc get pods -n openstack --no-headers | grep -v Completed | grep -v Running

# OpenStack endpoints
oc exec -t openstackclient -n openstack -- openstack endpoint list

# Verify compute services are online
oc exec -t openstackclient -n openstack -- openstack compute service list

# Verify network agents are up
oc exec -t openstackclient -n openstack -- openstack network agent list

# All resources labeled for restore (Secrets, ConfigMaps, NADs, CRs — everything)
oc get $(oc api-resources --verbs=list -o name | paste -sd, -) \
  -l backup.openstack.org/restore=true -n openstack 2>/dev/null
```

### Additional Checks

Depending on your environment, consider running further validation:

```bash
# Create a test instance to verify Nova, Neutron, Glance integration
oc exec -t openstackclient -n openstack -- openstack server create --flavor m1.small \
  --image cirros --network default test-post-restore

# Verify existing instances can be managed (stop, start, migrate)
oc exec -t openstackclient -n openstack -- openstack server list
oc exec -t openstackclient -n openstack -- openstack server stop <existing-instance>
oc exec -t openstackclient -n openstack -- openstack server start <existing-instance>
oc exec -t openstackclient -n openstack -- openstack server migrate <existing-instance>

# Run Tempest tests via the test-operator for comprehensive validation
oc apply -f <your-tempest-cr.yaml>
```

## ExtraMounts PVCs

PVCs passed via `extraMounts` are user-managed and must be labeled manually:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-extra-data
  namespace: openstack
  labels:
    backup.openstack.org/backup: "true"
    backup.openstack.org/restore: "true"
    backup.openstack.org/restore-order: "00"
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Customizing Restore Behavior

Override the controller's default labeling for individual resources using
annotations:

```bash
# Exclude a secret from restore
oc annotate secret my-temp-secret -n openstack \
  backup.openstack.org/restore="false"

# Force restore of an operator-managed cert secret
oc annotate secret cert-keystone-internal-svc -n openstack \
  backup.openstack.org/restore="true"

# Override restore order
oc annotate secret custom-ca-cert -n openstack \
  backup.openstack.org/restore-order="05"
```

## Limitations

- **Operator version must match** between source and target clusters
- **Namespace change not supported** — restore to the same namespace name
- **OVN DB backup is optional** and requires manually labeling the OVN PVCs
  for backup and running `ovsdb-client backup` (Step 3) before the OADP PVC
  backup. If skipped, the OVN databases will be empty after restore — see
  Step 10 (Sync Neutron to OVN).
- **VM network connectivity** may be interrupted during restore. Without
  OVN DB backup (Step 3 skipped), connectivity is lost when `ovn-controller`
  reconnects to the empty SB database and is restored by
  `neutron-ovn-db-sync-util` (Step 10).
- **RabbitMQ** is recreated as a fresh cluster with restored credentials
  (in-flight messages are lost)
- **Running VM state** reflects the backup point in time. If VMs were
  migrated after the backup, Nova's database will not match the actual
  placement.
- **Fully updated environments only** — backup/restore is not supported for
  partial update states
- **OpenStackBaremetalSet restore annotations** — restoring `OpenStackBaremetalSet`
  resources requires the `openstack.org/skip-webhook-validation` and
  `openstack.org/paused` annotations (injected by the resource modifier in
  Step 0). These annotations must be manually removed after restore (Step 13)
  to re-enable webhook validation and operator reconciliation

