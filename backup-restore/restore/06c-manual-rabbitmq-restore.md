# Order 55: Manual RabbitMQ Credential Restore

The new RabbitMQ clusters created during restore have random credentials.
This step restores the original `*-default-user` secrets from the backup
and creates RabbitMQUser CRs to re-establish the old credentials.

## Prerequisites

- Order 00-50 restore completed (database restore done)
- RabbitMQ clusters are running
- OADP backup containing the original `*-default-user` secrets is available

## Steps

### 1. Restore secrets to a temporary namespace

First, create a resource modifier ConfigMap that strips finalizers so
the temp namespace can be deleted cleanly:

```bash
oc create namespace openstack-restore-tmp

cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: openstack-restore-tmp-resource-modifiers
  namespace: openshift-adp
data:
  resource-modifiers.yaml: |
    version: v1
    resourceModifierRules:
    - conditions:
        groupResource: "*"
        namespaces:
        - openstack-restore-tmp
      mergePatches:
      - patchData: |
          metadata:
            finalizers: null
EOF
```

Then restore all secrets from the backup to the temp namespace. Replace
`RESOURCES_BACKUP` with your backup name (e.g.,
`openstack-backup-resources-20260311-081234`):

```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: openstack-restore-rabbitmq-secrets-${RESTORE_SUFFIX}
  namespace: openshift-adp
spec:
  backupName: ${RESOURCES_BACKUP}
  includedNamespaces:
  - openstack
  namespaceMapping:
    openstack: openstack-restore-tmp
  includedResources:
  - secrets
  resourceModifier:
    kind: ConfigMap
    name: openstack-restore-tmp-resource-modifiers
EOF

oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/openstack-restore-rabbitmq-secrets-${RESTORE_SUFFIX} -n openshift-adp --timeout=5m
```

### 2. Copy old credentials to target namespace

```bash
# Get RabbitMQ cluster names from the OpenStackControlPlane CR
RABBITMQ_CLUSTERS=$(oc get openstackcontrolplane -n openstack \
  -o jsonpath='{.items[0].spec.rabbitmq.templates}' | jq -r 'keys[]')

for CLUSTER in ${RABBITMQ_CLUSTERS}; do
  if ! oc get secret "${CLUSTER}-default-user" -n openstack-restore-tmp &>/dev/null; then
    echo "Secret ${CLUSTER}-default-user not found in temp namespace - skipping"
    continue
  fi

  TMPDIR=$(mktemp -d)
  oc extract secret/${CLUSTER}-default-user -n openstack-restore-tmp --to=${TMPDIR} --confirm
  oc create secret generic ${CLUSTER}-restored-user -n openstack --from-file=${TMPDIR}
  rm -rf ${TMPDIR}
  echo "Created ${CLUSTER}-restored-user in openstack"
done
```

### 3. Create RabbitMQUser CRs

```bash
for CLUSTER in ${RABBITMQ_CLUSTERS}; do
  RESTORED_SECRET="${CLUSTER}-restored-user"

  if ! oc get secret "${RESTORED_SECRET}" -n openstack &>/dev/null; then
    echo "Secret ${RESTORED_SECRET} not found - skipping"
    continue
  fi

  cat <<EOF | oc apply -f -
  apiVersion: rabbitmq.openstack.org/v1beta1
  kind: RabbitMQUser
  metadata:
    name: ${CLUSTER}-restored-user
    namespace: openstack
  spec:
    rabbitmqClusterName: ${CLUSTER}
    secret: ${RESTORED_SECRET}
    tags:
      - administrator
    permissions:
      configure: ".*"
      read: ".*"
      write: ".*"
EOF
  echo "Created RabbitMQUser CR for ${CLUSTER}"
done
```

### 4. Clean up temp namespace

```bash
oc delete namespace openstack-restore-tmp
```

## Next Steps

After RabbitMQ credential restore, return to the main restore procedure in
[README.md](README.md#step-6c-remove-deployment-stage-annotation) to
remove the deployment-stage annotation and resume full deployment.
