# Deploy MinIO on OpenShift

MinIO provides S3-compatible object storage for OADP backups. This guide
covers deploying MinIO on OpenShift. For OADP installation and configuration,
see [`../oadp/README.md`](../oadp/README.md).

## Prerequisites

- OpenShift cluster with cluster-admin access
- `oc` CLI tool installed and configured
- Sufficient storage for MinIO (PVs available in your cluster)

## Quick Start (ci-framework)

MinIO is deployed automatically as part of the dependency installation
step in the
[ci-framework](https://github.com/openstack-k8s-operators/ci-framework)
backup/restore playbook:

```bash
ansible-playbook playbooks/backup_restore.yaml \
  -e cifmw_backup_restore_install_deps=true \
  -e cifmw_backup_restore_run_backup=false \
  -e cifmw_backup_restore_run_cleanup=false \
  -e cifmw_backup_restore_run_restore=false
```

Configurable variables (see `roles/deploy_minio/defaults/main.yml`):

| Variable | Default | Description |
|----------|---------|-------------|
| `cifmw_deploy_minio_namespace` | `minio` | Namespace for MinIO |
| `cifmw_deploy_minio_storage_size` | `10Gi` | PVC size |
| `cifmw_deploy_minio_storage_class` | `""` (default) | StorageClass |
| `cifmw_deploy_minio_root_user` | `minio` | Root user |
| `cifmw_deploy_minio_root_password` | `minio123` | Root password |

## Manual Setup

### Step 1: Create MinIO Namespace

```bash
oc create namespace minio
```

### Step 2: Create MinIO Deployment

Create a file `minio-deployment.yaml`:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: minio
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Secret
metadata:
  name: minio-credentials
  namespace: minio
type: Opaque
stringData:
  MINIO_ROOT_USER: minio
  MINIO_ROOT_PASSWORD: minio123  # Change this in production!
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio
spec:
  selector:
    matchLabels:
      app: minio
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: quay.io/minio/minio:latest
        args:
        - server
        - /data
        - --console-address
        - :9001
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: MINIO_ROOT_USER
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: MINIO_ROOT_PASSWORD
        ports:
        - containerPort: 9000
          name: api
        - containerPort: 9001
          name: console
        volumeMounts:
        - name: data
          mountPath: /data
        livenessProbe:
          httpGet:
            path: /minio/health/live
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /minio/health/ready
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 20
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: minio
spec:
  type: ClusterIP
  ports:
  - port: 9000
    targetPort: 9000
    name: api
  - port: 9001
    targetPort: 9001
    name: console
  selector:
    app: minio
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio-console
  namespace: minio
spec:
  to:
    kind: Service
    name: minio
  port:
    targetPort: console
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio-api
  namespace: minio
spec:
  to:
    kind: Service
    name: minio
  port:
    targetPort: api
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

Deploy MinIO:

```bash
oc apply -f minio-deployment.yaml
```

### Step 3: Verify MinIO Deployment

```bash
oc wait --for=condition=available --timeout=300s deployment/minio -n minio
oc get pods -n minio
```

**MinIO Web Console:**

```bash
echo "MinIO Console: https://$(oc get route minio-console -n minio -o jsonpath='{.spec.host}')"
echo "Credentials: minio / minio123"
```

### Step 4: Create Backup Bucket

```bash
# Download mc
curl -o /tmp/mc https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x /tmp/mc

# Configure mc to use your MinIO instance
MINIO_API_URL=$(oc get route minio-api -n minio -o jsonpath='{.spec.host}')
/tmp/mc alias set minio https://${MINIO_API_URL} minio minio123

# Create bucket
/tmp/mc mb minio/velero
```

### Step 5: Create MinIO Service Account for OADP

```bash
# This creates a service account with full access
# Save the output Access Key and Secret Key
MINIO_API_URL=$(oc get route minio-api -n minio -o jsonpath='{.spec.host}')
/tmp/mc admin user svcacct add minio minio
```

**Important**: Save the generated **Access Key** and **Secret Key** — you'll
need these for [OADP configuration](../oadp/README.md).

## Verifying Backups in MinIO

### Using MinIO Web Console (GUI)

```bash
echo "MinIO Console: https://$(oc get route minio-console -n minio -o jsonpath='{.spec.host}')"
```

Browse to **Buckets** > **velero** to see backup data:

```
velero/
└── rhoso/
    ├── backups/
    │   └── openstack-backup-pvcs-YYYYMMDD-HHMMSS/
    └── kopia/
        └── openstack/
```

### Using MinIO Client (mc) CLI

```bash
MINIO_ENDPOINT=$(oc get route minio-api -n minio -o jsonpath='{.spec.host}')
mc alias set minio-oadp https://${MINIO_ENDPOINT} minio minio123 --insecure

# List all backups
mc ls minio-oadp/velero/rhoso/backups/ --insecure

# Check backup size
mc du minio-oadp/velero/rhoso/backups/<backup-name>/ --insecure

# Check total storage used
mc du minio-oadp/velero/ --insecure
```

### Using oc exec into MinIO Pod

```bash
oc exec -it -n minio deployment/minio -- sh -c '
  mc alias set local http://localhost:9000 minio minio123
  mc ls local/velero/rhoso/backups/
'
```

## Troubleshooting

### MinIO Storage Full

```bash
oc get pvc -n minio
# Expand PVC if needed:
oc patch pvc minio-pvc -n minio -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

## Production Recommendations

1. **External MinIO**: Consider running MinIO outside the cluster or using managed S3
2. **High Availability**: Deploy MinIO in distributed mode
3. **Persistent Storage**: Use high-performance storage class
4. **Strong Credentials**: Use strong passwords and rotate regularly
5. **TLS Certificates**: Configure proper TLS instead of insecure skip verify

## See Also

- OADP setup: [`../oadp/README.md`](../oadp/README.md)
- [MinIO Documentation](https://min.io/docs/minio/kubernetes/upstream/)
