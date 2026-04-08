# Install and Configure OADP on OpenShift

OADP (OpenShift API for Data Protection) provides backup and restore
capabilities using Velero. This guide covers installing OADP and configuring
it with a MinIO backend. For MinIO deployment, see
[`../minio/README.md`](../minio/README.md).

## Prerequisites

- OpenShift cluster with cluster-admin access
- MinIO deployed and accessible (see [`../minio/README.md`](../minio/README.md))
- MinIO service account credentials (Access Key and Secret Key)
- `oc` CLI tool installed and configured

## Quick Start (ci-framework)

OADP is installed automatically as part of the dependency installation
step in the
[ci-framework](https://github.com/openstack-k8s-operators/ci-framework)
backup/restore playbook (after MinIO):

```bash
ansible-playbook playbooks/backup_restore.yaml \
  -e cifmw_backup_restore_install_deps=true \
  -e cifmw_backup_restore_run_backup=false \
  -e cifmw_backup_restore_run_cleanup=false \
  -e cifmw_backup_restore_run_restore=false
```

Configurable variables (see `roles/openshift_adp/defaults/main.yml`):

| Variable | Default | Description |
|----------|---------|-------------|
| `cifmw_openshift_adp_namespace` | `openshift-adp` | OADP namespace |
| `cifmw_openshift_adp_channel` | `stable` | OLM subscription channel |
| `cifmw_openshift_adp_enable_node_agent` | `true` | Enable Kopia node agent (Data Mover) |
| `cifmw_openshift_adp_s3_bucket` | `velero` | S3 bucket name |
| `cifmw_openshift_adp_s3_prefix` | `rhoso` | Bucket prefix |
| `cifmw_openshift_adp_s3_insecure_skip_tls` | `true` | Skip TLS verification |

The S3 credentials (`cifmw_openshift_adp_s3_access_key`,
`cifmw_openshift_adp_s3_secret_key`) are passed automatically from the
`deploy_minio` role output.

## Manual Setup

### Step 1: Install OADP Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-adp
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-adp-operator-group
  namespace: openshift-adp
spec:
  targetNamespaces:
  - openshift-adp
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redhat-oadp-operator
  namespace: openshift-adp
spec:
  channel: stable
  installPlanApproval: Automatic
  name: redhat-oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Wait for the operator:

```bash
oc wait --for=condition=ready --timeout=300s pod -l control-plane=controller-manager -n openshift-adp
```

### Step 2: Create Cloud Credentials Secret

```bash
cat <<EOF > /tmp/credentials-velero
[default]
aws_access_key_id=<MINIO_ACCESS_KEY>
aws_secret_access_key=<MINIO_SECRET_KEY>
EOF

oc create secret generic cloud-credentials \
  --from-file cloud=/tmp/credentials-velero \
  -n openshift-adp

rm /tmp/credentials-velero
```

Replace `<MINIO_ACCESS_KEY>` and `<MINIO_SECRET_KEY>` with the credentials
from the MinIO service account.

### Step 3: Create DataProtectionApplication (DPA)

```bash
MINIO_API_URL=$(oc get route minio-api -n minio -o jsonpath='{.spec.host}')

cat <<EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: velero
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
      - openshift
      - aws
      - csi
    nodeAgent:
      enable: true
      uploaderType: kopia
  backupLocations:
  - velero:
      provider: aws
      default: true
      objectStorage:
        bucket: velero
        prefix: rhoso
      config:
        region: minio
        s3ForcePathStyle: "true"
        s3Url: https://${MINIO_API_URL}
        insecureSkipTLSVerify: "true"
      credential:
        name: cloud-credentials
        key: cloud
EOF
```

**Important Configuration Notes:**

- `s3ForcePathStyle: "true"` — required for MinIO
- `insecureSkipTLSVerify: "true"` — MinIO route uses OpenShift self-signed
  certificate. In production, configure proper TLS.
- **defaultPlugins includes `csi`** — REQUIRED for CSI volume snapshots.
  Without it, OADP will not create VolumeSnapshots. The three plugins:
  - `openshift` — OpenShift-specific resources
  - `aws` — S3 backup storage (works with MinIO)
  - `csi` — CSI volume snapshots (critical for OpenStack backups)
- **snapshotLocations is NOT included** — intentional! For CSI volume
  snapshots (LVMS/TopoLVM, Ceph RBD), you must NOT configure
  `snapshotLocations`. That field is only for cloud provider snapshots
  (AWS EBS, Azure Disk, GCP PD).
- **Node Agent (Kopia)** — enabled for Data Mover support. When
  `snapshotMoveData: true` is set on a Backup CR, Kopia uploads CSI
  snapshot data to MinIO, enabling restore even after cluster loss.

### Step 4: Verify Installation

```bash
# Check pods
oc get pods -n openshift-adp
# Expected: openshift-adp-controller-manager-*, velero-*, node-agent-* (one per node)

# Check BackupStorageLocation
oc get backupstoragelocation -n openshift-adp
# Status should show Available

# Check Velero logs
oc logs -n openshift-adp deployment/velero
```

### Checking Velero Version

```bash
VELERO_POD=$(oc get pods -n openshift-adp -l deploy=velero \
  --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')
oc exec -n openshift-adp ${VELERO_POD} -- /velero version
```

OCP 4.20 ships OADP 1.5 with Velero v1.16+, which adds the
`ignoreDelayBinding` flag for node-agent, improving Data Mover handling
of `WaitForFirstConsumer` PVCs. On OCP 4.18/4.19 with older OADP
versions, see [`../restore/README.md`](../restore/README.md) for
workarounds.

## Test Backup and Restore (Optional)

### Create Test Application

```bash
oc create namespace test-backup
oc run nginx --image=nginx -n test-backup
oc expose pod nginx --port=80 -n test-backup
oc create configmap test-data --from-literal=key=value -n test-backup
```

### Create Backup

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: test-backup
  namespace: openshift-adp
spec:
  includedNamespaces:
  - test-backup
  snapshotMoveData: true
  storageLocation: velero-1
EOF

oc get backup -n openshift-adp -w
```

### Restore

```bash
oc delete namespace test-backup

cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: test-restore
  namespace: openshift-adp
spec:
  backupName: test-backup
EOF

oc get restore -n openshift-adp -w
oc get all -n test-backup
```

### Clean Up

```bash
oc delete namespace test-backup
oc delete backup test-backup -n openshift-adp
oc delete restore test-restore -n openshift-adp
```

## Troubleshooting

### BackupStorageLocation Shows "Unavailable"

```bash
oc logs -n openshift-adp deployment/velero
```

Common issues:
- **Authentication failed**: Verify cloud-credentials secret
- **Connection refused**: Verify MinIO API URL
- **TLS errors**: Check certificate configuration
- **Bucket not found**: Verify the `velero` bucket exists in MinIO

### Node Agent Pods Not Running

```bash
oc get daemonset node-agent -n openshift-adp -o yaml
```

## Production Recommendations

1. **Multiple Backup Locations**: Configure redundant backup locations
2. **Backup Scheduling**:
   ```yaml
   apiVersion: velero.io/v1
   kind: Schedule
   metadata:
     name: daily-backup
     namespace: openshift-adp
   spec:
     schedule: "0 2 * * *"
     template:
       includedNamespaces:
       - openstack
       snapshotMoveData: true
   ```
3. **Backup Retention**: Configure TTL (`ttl: 720h` = 30 days)
4. **Resource Limits**: Set limits for Velero and node-agent
5. **Monitoring**: Set up alerts for backup failures

## References

- [OADP Documentation](https://docs.openshift.com/container-platform/latest/backup_and_restore/application_backup_and_restore/oadp-intro.html)
- [Velero Documentation](https://velero.io/docs/)

## See Also

- MinIO setup: [`../minio/README.md`](../minio/README.md)
- Backup CRs: [`../backup/README.md`](../backup/README.md)
- Restore CRs: [`../restore/README.md`](../restore/README.md)
