# OpenStack Restore Process

This document describes the OADP Restore CR strategy and manual restore
procedure for OpenStack environments. The restore playbooks and templates
are in the [ci-framework](https://github.com/openstack-k8s-operators/ci-framework)
`cifmw_backup_restore` role.

## Restore Order

Restores must be executed in sequence. Wait for each restore to complete
before starting the next.

| Order | Resources | Notes |
|-------|-----------|-------|
| - | ConfigMap | **Prerequisite:** resource modifier rules for all restores |
| 00 | PVCs (Glance images, GaleraBackup) | Storage foundation, restored from CSI snapshots |
| 10 | Secrets, ConfigMaps, NADs | User-provided resources without ownerRefs |
| 20 | OpenStackVersion, OpenStackBackupConfig, Issuers, NetConfig, Topology, BGPConfiguration, DNSData, InstanceHa | Infrastructure base (**InstanceHa restored with `spec.disabled: True`**) |
| 30 | OpenStackControlPlane, Reservation | **Restored with `deployment-stage: infrastructure-only` annotation** |
| 40 | GaleraBackup, IPSet, DataPlaneService | Backup config, IP sets, custom DataPlane services |
| 50 | **Manual**: Database restore | Create GaleraRestore CRs, restore databases, remove deployment-stage annotation, restart the Nova pods (Step 6c) |
| 55 | BaremetalHosts (Step 7), OpenStackBaremetalSet (Step 8) | Optional: baremetal-provisioned nodes only. BMH secrets/CMs restored here if separate namespace. Requires BMH pause annotation, OSBMS webhook-skip and pause annotations (resource modifier), and `restoreStatus` |
| 60 | OpenStackDataPlaneNodeSet | DataPlane resources (optional) |
| - | OpenStackDataPlaneDeployment | Resync credentials on dataplane nodes |
| - | **Manual** | Re-enable InstanceHa (`spec.disabled: False`) after verifying the cloud is operational |

## Prerequisites

- OADP operator installed and configured
- Velero BackupStorageLocation (BSL) configured and pointing to the same
  object storage (S3/MinIO) used during backup
- CSI snapshot capability for PVC restores
- Target namespace exists (or will be created during restore)

### Velero Version

```bash
VELERO_POD=$(oc get pods -n openshift-adp -l deploy=velero \
  --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')
oc exec -n openshift-adp ${VELERO_POD} -- /velero version
```

OCP 4.20 ships OADP 1.5 with Velero v1.16+, which adds the
`ignoreDelayBinding` flag for node-agent, improving Data Mover handling
of `WaitForFirstConsumer` PVCs. On OCP 4.18/4.19 with older OADP
versions, see [Data Mover with WaitForFirstConsumer](#data-mover-restore-with-waitforfirstconsumer-storage-lvm)
for workarounds.

### Discovering Backups

When using the Data Mover (`snapshotMoveData: true`), backup metadata and PVC
data are stored in the BackupStorageLocation (S3/MinIO). This enables restore
even on a completely new cluster — Velero automatically syncs and discovers
existing backups from the object storage.

```bash
# 1. Verify BSL is available and syncing
oc get backupstoragelocation -n openshift-adp

# 2. Wait for Velero to sync (default: every 1 minute)
oc get backup -n openshift-adp

# 3. Find the backup names to use for restore
oc get backup -n openshift-adp -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,CREATED:.metadata.creationTimestamp
```

## Quick Start (ci-framework)

Use the ci-framework restore playbook (see [top-level README](../README.md)):

```bash
# Cleanup + restore (backup already done):
ansible-playbook playbooks/backup_restore.yaml \
  -e cifmw_backup_restore_install_deps=false \
  -e cifmw_backup_restore_run_backup=false \
  -e cifmw_backup_restore_run_cleanup=true \
  -e cifmw_backup_restore_run_restore=true \
  -e cifmw_backup_restore_backup_name_suffix=20260311-081234
```

The playbook runs all restore steps in order, including:
- Ordered OADP restores (PVCs > Foundation > Infrastructure > ControlPlane > GaleraBackup)
- Waits for infrastructure to be ready (Galera, OVN, RabbitMQ)
- Automated database restore (creates GaleraRestore CRs, runs restore script)
- Removes deployment-stage annotation to resume full deployment
- Waits for control plane to be ready
- Restarts the Nova control plane service pods so they re-read the restored cell mappings
  (`cifmw_backup_restore_restart_nova`, default `true`)
- Optional DataPlane restore
- EDPM deployment to resync credentials on dataplane nodes

## Manual Restore Procedure

If you are not using the ci-framework playbooks, follow these steps manually.

Set variables used throughout:

```bash
BACKUP_TIMESTAMP=20260311-081234
RESTORE_SUFFIX=$(date +%Y%m%d-%H%M%S)
PVC_BACKUP=openstack-backup-pvcs-${BACKUP_TIMESTAMP}
RESOURCES_BACKUP=openstack-backup-resources-${BACKUP_TIMESTAMP}
```

### Pre-check: Verify Operator Version

The target cluster must have the **same operator version** as the source
cluster. Retrieve the required version from the backup annotations and
compare with the installed version:

```bash
# Get required version from backup
echo "=== Required operator version (from backup) ==="
oc get backup ${RESOURCES_BACKUP} -n openshift-adp -o custom-columns=\
CSV:.metadata.annotations.openstack\.org/csv-version,\
CATALOG:.metadata.annotations.openstack\.org/catalog-source-image,\
OPERATOR:.metadata.annotations.openstack\.org/operator-image

# Get installed version
echo "=== Installed operator version ==="
oc get csv -n openstack-operators \
  -l operators.coreos.com/openstack-operator.openstack-operators \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase
```

If the versions do not match, install the correct operator version before
proceeding. For custom builds, use the catalog source image from the backup
annotations. See [Operator Version Must Match](../README.md#operator-version-must-match)
for details.

### Step 0: Create Resource Modifier ConfigMap

This ConfigMap is referenced by all Restore CRs. It strips
`last-applied-configuration` annotations, adds the `deployment-stage`
annotation to the ControlPlane CR, disables InstanceHa during restore,
and pauses any BaremetalHosts to prevent Metal3 from acting on them
before their status is restored.

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
    - conditions:
        groupResource: "*"
      mergePatches:
      - patchData: |
          metadata:
            ownerReferences: null
            annotations:
              kubectl.kubernetes.io/last-applied-configuration: null
    - conditions:
        groupResource: openstackcontrolplanes.core.openstack.org
      mergePatches:
      - patchData: |
          metadata:
            annotations:
              kubectl.kubernetes.io/last-applied-configuration: null
              core.openstack.org/deployment-stage: "infrastructure-only"
    - conditions:
        groupResource: instancehas.instanceha.openstack.org
      mergePatches:
      - patchData: |
          spec:
            disabled: "True"
    # BaremetalHosts — pause BMHs during restore so Metal3 does not act
    # on them before their status is restored.
    - conditions:
        groupResource: baremetalhosts.metal3.io
      mergePatches:
      - patchData: |
          metadata:
            annotations:
              baremetalhost.metal3.io/paused: ""
    # OpenStackBaremetalSets — skip webhook validation and pause
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

### Step 0.5: Pin PVCs (optional, LVM with Data Mover only)

**Required for LVM/topolvm environments using Data Mover on OCP < 4.20.**
See [Data Mover with WaitForFirstConsumer](#data-mover-restore-with-waitforfirstconsumer-storage-lvm)
below.

### Step 1: Restore PVCs (Order 00)

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-00-pvcs-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${PVC_BACKUP}
  includedNamespaces:
  - openstack
  excludedResources:
  - pods
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
  restorePVs: true
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-00-pvcs-${RESTORE_SUFFIX} -n openshift-adp --timeout=15m
```

If step 0.5 was used, delete dummy Deployments now:
```bash
oc delete deployment -n openstack -l app=pvc-pin
```

### Step 2: Restore Foundation Resources (Order 10)

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-10-foundation-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${RESOURCES_BACKUP}
  includedNamespaces:
  - openstack
  labelSelector:
    matchLabels:
      backup.openstack.org/restore: "true"
      backup.openstack.org/restore-order: "10"
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-10-foundation-${RESTORE_SUFFIX} -n openshift-adp --timeout=5m
```

### Step 3: Restore Infrastructure CRs (Order 20)

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

### Step 4: Restore OpenStackControlPlane (Order 30)

The resource modifier adds `deployment-stage: infrastructure-only` so
services don't start until databases are restored.

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

### Step 5: Restore Backup Config (Order 40)

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

### Step 6: Manual Steps (Order 50+)

These steps require manual intervention and must be performed in order.

#### Step 6a: Database Restore

Follow the detailed procedure in
[06-manual-database-restore.md](06-manual-database-restore.md).

Summary:
1. Create GaleraRestore CRs for each GaleraBackup
2. Wait for restore pods
3. Execute `restore_galera` inside each pod
4. Delete GaleraRestore CRs

#### Step 6b: Remove deployment-stage Annotation

After database restore is complete, remove the
`deployment-stage` annotation to resume full OpenStack deployment.
**This is critical** — without it, services remain in infrastructure-only
mode and will not start.

```bash
oc patch openstackcontrolplane \
  $(oc get openstackcontrolplane -n openstack -o jsonpath='{.items[0].metadata.name}') \
  -n openstack --type json \
  -p '[{"op": "remove", "path": "/metadata/annotations/core.openstack.org~1deployment-stage"}]'
```

Wait for all OpenStack services to start:

```bash
oc wait openstackcontrolplane -n openstack --for=condition=Ready --timeout=30m \
  $(oc get openstackcontrolplane -n openstack -o jsonpath='{.items[0].metadata.name}')
```

#### Step 6c: Restart the Nova Services

The restored `nova_api.cell_mappings` carry the cell database credentials from
before the restore, while the operator creates new cell database accounts.
Nova services that started in that window, notably the super-conductor, cache
connections that use the removed database users, so new instances scheduled to
the affected cells hang in `BUILD`. Delete the Nova control plane service pods
so they are recreated and re-read the reconciled mappings. The selector matches
the API, scheduler, metadata, conductor and novncproxy pods; the API and
scheduler pods are not part of a cell, so this restarts all of the Nova control
plane services:

```bash
oc delete pod -n openstack \
  -l 'service in (nova-api,nova-scheduler,nova-metadata,nova-conductor,nova-novncproxy)' \
  --field-selector=status.phase=Running
```

Only `Running` pods are selected because the Nova DB purge CronJob pods share
the `service=nova-conductor` label and remain `Completed`.

Wait for the recreated pods to be `Ready` and for the control plane to be
`Ready` again:

```bash
oc get pods -n openstack \
  -l 'service in (nova-api,nova-scheduler,nova-metadata,nova-conductor,nova-novncproxy)'

oc wait openstackcontrolplane -n openstack --for=condition=Ready --timeout=30m \
  $(oc get openstackcontrolplane -n openstack -o jsonpath='{.items[0].metadata.name}')
```

The listing deliberately has no `Running` filter, so that replacement pods that
are still `Pending` or not yet `Ready` stay visible. Every service, and every
cell on multi-cell deployments, should show its expected number of `Ready`
pods. Completed Nova DB purge pods may also be listed; ignore them.

### Step 7: Restore BaremetalHosts (Order 55, optional — baremetal-provisioned only)

Skip this step if all OpenStackDataPlaneNodeSets use `preProvisioned: true`.

The resource modifier from Step 0 injects a
`baremetalhost.metal3.io/paused` annotation so Metal3 does not act on
the BMHs before their status is restored.

**Namespace considerations:**

- If your BaremetalHosts are in the `openstack` namespace, their secrets
  and configmaps are already restored in Step 2 (order 10). Skip the
  secrets/configmaps restore below and only restore the BaremetalHosts.
  Use `${RESOURCES_BACKUP}` as the `backupName` and set
  `includedNamespaces` to `openstack`.
- If your BaremetalHosts are in a **separate namespace**, restore their
  secrets and configmaps first from the BMH backup (Backup Step 5),
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

Remove pause annotations. Consider unpausing a single BMH first to
verify it reaches the expected state before unpausing the rest:

```bash
oc annotate bmh -n ${BMH_NAMESPACE} <single-bmh-name> baremetalhost.metal3.io/paused-
oc get bmh -n ${BMH_NAMESPACE} <single-bmh-name>

# After confirming the BMH is in the expected state, unpause the rest
oc annotate bmh -n ${BMH_NAMESPACE} --all baremetalhost.metal3.io/paused-
oc get bmh -n ${BMH_NAMESPACE}
```

### Step 8: Restore OpenStackBaremetalSets (Order 55, optional — baremetal-provisioned only)

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

The `OpenStackBaremetalSet` will temporarily leave `Ready` state while
`OpenStackProvisionServer` reconciles, but BMHs and their workloads are
unaffected.

### Step 9: Restore DataPlane (Order 60, optional)

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

### Step 10: EDPM Deployment

Resync credentials on dataplane nodes:

```bash
NODESETS=$(oc get openstackdataplanenodeset -n openstack -o jsonpath='{.items[*].metadata.name}')
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

### Step 11: Re-enable InstanceHa

After verifying the restored cloud is fully operational:

```bash
oc patch instanceha <name> -n openstack --type merge -p '{"spec":{"disabled":"False"}}'
```

## Data Mover Restore with WaitForFirstConsumer Storage (LVM)

**Required for LVM/topolvm environments using Data Mover on OCP < 4.20.**
Skip if not using Data Mover (`snapshotMoveData: true`) or if your
StorageClass uses `volumeBindingMode: Immediate`.

When using the OADP Data Mover with a StorageClass that has
`volumeBindingMode: WaitForFirstConsumer` (e.g., LVM/topolvm), the PVC restore
will deadlock. The data mover waits for the PVC to be consumed by a pod before
downloading data, but with WaitForFirstConsumer the PVC won't bind until a pod
references it. Since PVCs are restored before any workload pods exist, this
creates a deadlock.

**Solution:** Pre-create dummy Deployments with `nodeSelector` to pin PVCs to
their original nodes. The PVC-to-node mapping is extracted from the PV
`nodeAffinity` in the backup metadata.

Use `nodeSelector` (not `nodeName`) because `nodeName` bypasses the scheduler
and the PVC `volume.kubernetes.io/selected-node` annotation never gets set.

```bash
# 1. Download the backup metadata (small — only resource manifests, no PV data)
velero backup download <pvc-backup-name> -o /tmp/backup.tar.gz
mkdir -p /tmp/backup && tar xzf /tmp/backup.tar.gz -C /tmp/backup

# 2. Extract PVC-to-node mapping from PVs (topolvm stores node in nodeAffinity)
for f in /tmp/backup/resources/persistentvolumes/cluster/*.json; do
  pvc=$(jq -r '.spec.claimRef.name // empty' "$f")
  ns=$(jq -r '.spec.claimRef.namespace // empty' "$f")
  node=$(jq -r '
    .spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[]
    | select(.key | contains("topolvm")) | .values[0]
  ' "$f" 2>/dev/null)
  [ -n "$pvc" ] && [ -n "$node" ] && echo "${pvc}:${node}"
done
# Example output:
#   glance-glance-default-single-0:master-0
#   mysql-db-openstack-galera-0:master-0

# 3. Create dummy Deployments BEFORE the PVC restore (step 1).
#    IMPORTANT: Use nodeSelector (not nodeName) — see above.
for pvc_node in \
  "glance-glance-default-single-0:master-0" \
  "mysql-db-openstack-galera-0:master-0"; do

  pvc="${pvc_node%%:*}"
  node="${pvc_node##*:}"

  cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-pin-${pvc}
  namespace: openstack
  labels:
    app: pvc-pin
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: pvc-pin
      pvc: ${pvc}
  template:
    metadata:
      labels:
        app: pvc-pin
        pvc: ${pvc}
    spec:
      nodeSelector:
        kubernetes.io/hostname: ${node}
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: ${pvc}
EOF
done

# 4. Continue with the PVC restore (step 1)
# 5. After PVC restore completes, delete dummy Deployments:
oc delete deployment -n openstack -l app=pvc-pin
```

**Upstream issues:**
- [velero#7561](https://github.com/vmware-tanzu/velero/issues/7561) — WaitForFirstConsumer incompatibility
- [velero#8044](https://github.com/vmware-tanzu/velero/issues/8044) — Enhancement proposal
- [velero#9343](https://github.com/vmware-tanzu/velero/issues/9343) — Topology-aware storage

## Verification

After each restore step:

```bash
# Check restore status
oc get restore -n openshift-adp

# Check restored resources
oc get all,pvc,secret,configmap,nad -n openstack

# Check OpenStack CRs
oc get openstackcontrolplane,openstackversion -n openstack
```

After the restore is complete, boot a new instance in every available compute
availability zone and verify that it reaches `ACTIVE`. This exercises scheduling
and the cell database write path, which stopping and starting an existing
instance does not. The ci-framework role does this automatically when the test
workload is enabled (`cifmw_backup_restore_validate_az_boot`, default `true`).
See [Additional Checks](../user-guide.md#additional-checks) in the user guide.

Availability zones are used as a proxy for Nova cells, so this check exercises
every cell only if each zone maps to exactly one cell. If several cells share a
zone, only one of them is tested. The ci-framework role prints a warning when
the `OpenStackControlPlane` defines more cells (excluding `cell0`) than there
are availability zones.

## Troubleshooting

### Restore stuck or failed

```bash
# Check restore details
oc describe restore <restore-name> -n openshift-adp

# Check Velero logs
oc logs -n openshift-adp deployment/velero
```

### PVC restore issues

```bash
# Check volume snapshots
oc get volumesnapshot -n openstack
oc get volumesnapshotcontent

# Check PVC events
oc describe pvc <pvc-name> -n openstack
```

### New instances stuck in BUILD after restore

Existing instances work, but newly created instances stay in `BUILD`. The Nova
super-conductor is most likely using cached cell database connections with
credentials that no longer exist. Restart the Nova control plane service pods (Step 6c):

```bash
oc delete pod -n openstack \
  -l 'service in (nova-api,nova-scheduler,nova-metadata,nova-conductor,nova-novncproxy)' \
  --field-selector=status.phase=Running
```

### Database restore issues

```bash
# Check GaleraRestore CR
oc get galerarestore -n openstack
oc describe galerarestore <name> -n openstack

# Check restore pod logs
oc logs <backup-source>-restore-<restore-name> -n openstack
```

## See Also

- Backup: [`../backup/README.md`](../backup/README.md)
- Manual database restore: [`06-manual-database-restore.md`](06-manual-database-restore.md)
- Design document: [`../backup-restore-controller-design.md`](../backup-restore-controller-design.md)
