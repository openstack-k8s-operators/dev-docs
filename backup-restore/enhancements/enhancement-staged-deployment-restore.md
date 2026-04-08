# Enhancement: Staged Deployment for Backup/Restore Scenarios

## Metadata

- **Status**: Implemented
- **Author**: OpenStack Operator Development Team
- **Created**: 2026-01-23
- **Implemented**: 2026-02-10 via [PR #1785](https://github.com/openstack-k8s-operators/openstack-operator/pull/1785)
- **Related Documentation**:
  - [backup-restore-ctlplane.md](backup-restore-ctlplane.md)

**Note**: This document is kept as historical reference and context for the staged deployment feature. The feature is now part of the main codebase.

## Summary

Add support for staged deployment to the OpenStackControlPlane controller, allowing infrastructure services (MariaDB, OVN, RabbitMQ, Memcached) to be created and paused before OpenStack services are deployed. This enables database restore to occur before services start, resulting in a cleaner and more reliable restore process.

**Key Benefits:**
- Services start with **already-restored databases** (no restarts needed)
- **Single condition check** in restore scripts: `oc wait --for=condition=InfrastructureReady` (controller validates all components)
- No schema conflicts or service errors during restore
- Simpler, more reliable restore workflow

**Implementation complexity: Low**
- Single if-statement added at line ~337 in controller
- Check annotation, validate infrastructure readiness, set status conditions, return early
- ~50 lines of code total (including readiness checks, status updated by defer function)
- 1-2 days of development + testing

## Motivation

### Problem Statement

Currently, when restoring an OpenStackControlPlane from backup, the restore workflow is:

1. Restore OpenStackControlPlane CR (and related resources)
2. Operators create ALL services simultaneously (MariaDB, OVN, Keystone, Nova, etc.)
3. Services start with **empty databases**
4. Wait for database pods to be ready
5. Restore database contents into running databases
6. Services must reconnect/restart to see restored data

**Issues with this approach:**

1. **Service startup with empty databases**: Services like Keystone, Nova start before databases are restored
2. **Schema initialization conflicts**: Services may run migrations or create initial schemas before restore
3. **Service restarts required**: After database restore, services need to restart to pick up the restored data
4. **Race conditions**: Timing between service startup and database restore is unpredictable
5. **Operational complexity**: Requires careful orchestration and monitoring
6. **Error handling**: Services may enter error states waiting for database schemas

### Desired Outcome

A staged deployment mode where:

1. Database infrastructure (MariaDB, OVN, RabbitMQ, Memcached) is created first
2. Deployment pauses automatically at line ~337 (perfect existing pause point)
3. Administrator restores database contents
4. Deployment resumes and creates remaining services
5. **Services start with fully-restored databases from the beginning**
6. No service restarts or reconnections needed

**Key insight:** The controller already reconciles infrastructure before services. We just need to add a check at line ~337 to pause there if annotation is set. Infrastructure is already ready at that point - OVN, RabbitMQ, Galera, and Memcached are all reconciled.

## Proposal

### Overview

Add a deployment control mechanism to OpenStackControlPlane that allows pausing the deployment after database infrastructure is ready but before application services are created.

### API Changes

Use annotations for runtime control without API changes:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
  namespace: openstack
  annotations:
    # Pause deployment after infrastructure is ready
    core.openstack.org/deployment-stage: "infrastructure-only"
spec:
  # ... normal spec
```

**Annotation Key:**
- `core.openstack.org/deployment-stage`

**Values:**
- `infrastructure-only` - Deploy infrastructure (MariaDB, OVN, RabbitMQ, Memcached) only, then pause
- annotation removed (or not set) - Deploy all services (default behavior)

**Benefits:**
- No API version changes needed
- Runtime control without spec modification
- Easy to toggle during restore
- Follows Kubernetes conventions (similar to `kubectl rollout pause`)
- Can be added/removed without editing the main CR spec
- **Single condition to wait for**: Restore scripts only need to wait for `InfrastructureReady` condition instead of checking multiple individual components (Galera, OVN NB, OVN SB, RabbitMQ, Memcached)

### Implementation Details

#### Controller Logic

The implementation is straightforward - add a pause point after infrastructure reconciliation.

**Current code location:**
https://github.com/openstack-k8s-operators/openstack-operator/blob/main/internal/controller/core/openstackcontrolplane_controller.go#L337

At line ~337, the controller has already reconciled:
- OVN databases
- RabbitMQ
- Galera (MariaDB)
- Memcached

**Proposed change:**

```go
// Around line 337 in openstackcontrolplane_controller.go
// After reconciling infrastructure (OVN, RabbitMQ, Galera, Memcached)
// and before reconciling OpenStack services (Keystone, Nova, etc.)

// Check if we should pause for infrastructure restore
if instance.Annotations["core.openstack.org/deployment-stage"] == "infrastructure-only" {
    // Check if all infrastructure components are ready
    infraReady := true
    var notReadyComponents []string

    // Check Galera
    if instance.Spec.Galera.Enabled {
        galera := &mariadbv1.Galera{}
        if err := r.Get(ctx, types.NamespacedName{Name: "galera", Namespace: instance.Namespace}, galera); err != nil || !galera.Status.Conditions.IsTrue(condition.ReadyCondition) {
            infraReady = false
            notReadyComponents = append(notReadyComponents, "Galera")
        }
    }

    // Check OVN databases
    if instance.Spec.Ovn.Enabled {
        ovnNB := &ovnv1.OVNDBCluster{}
        if err := r.Get(ctx, types.NamespacedName{Name: "ovndbcluster-nb", Namespace: instance.Namespace}, ovnNB); err != nil || !ovnNB.Status.Conditions.IsTrue(condition.ReadyCondition) {
            infraReady = false
            notReadyComponents = append(notReadyComponents, "OVN-NB")
        }

        ovnSB := &ovnv1.OVNDBCluster{}
        if err := r.Get(ctx, types.NamespacedName{Name: "ovndbcluster-sb", Namespace: instance.Namespace}, ovnSB); err != nil || !ovnSB.Status.Conditions.IsTrue(condition.ReadyCondition) {
            infraReady = false
            notReadyComponents = append(notReadyComponents, "OVN-SB")
        }
    }

    // Check RabbitMQ
    // TODO: Add RabbitMQ readiness check

    // Check Memcached
    // TODO: Add Memcached readiness check

    if infraReady {
        // All infrastructure ready - set condition and pause
        instance.Status.Conditions.Set(condition.TrueCondition(
            corev1beta1.OpenStackControlPlaneInfrastructureReadyCondition,
            condition.ReadyMessage,
            "Infrastructure (Galera, OVN, RabbitMQ, Memcached) is ready for database restore. "+
            "Remove annotation core.openstack.org/deployment-stage to resume deployment.",
        ))

        instance.Status.Conditions.Set(condition.FalseCondition(
            condition.ReadyCondition,
            "DeploymentPaused",
            condition.SeverityInfo,
            "Deployment paused at infrastructure-only stage for restore",
        ))
    } else {
        // Infrastructure not ready yet - set waiting condition
        instance.Status.Conditions.Set(condition.FalseCondition(
            corev1beta1.OpenStackControlPlaneInfrastructureReadyCondition,
            "InfrastructureNotReady",
            condition.SeverityInfo,
            fmt.Sprintf("Waiting for infrastructure components to be ready: %s", strings.Join(notReadyComponents, ", ")),
        ))

        instance.Status.Conditions.Set(condition.FalseCondition(
            condition.ReadyCondition,
            "DeploymentPaused",
            condition.SeverityInfo,
            "Deployment paused at infrastructure-only stage, waiting for infrastructure",
        ))
    }

    // Status will be updated by defer function
    // Pause here - don't continue to service reconciliation
    return ctrl.Result{}, nil
}

// Continue with normal service reconciliation (Keystone, Nova, etc.)
// ...
```

#### Restore Script Simplification

**Key Benefit:** The `InfrastructureReady` condition dramatically simplifies restore scripts!

**Before (without this enhancement):**
```bash
# Must wait for each component individually
oc wait --for=condition=Ready pod -l component=galera -n openstack --timeout=300s
oc wait --for=condition=Ready pod/ovsdbserver-nb-0 -n openstack --timeout=300s
oc wait --for=condition=Ready pod/ovsdbserver-sb-0 -n openstack --timeout=300s
oc wait --for=condition=Ready pod -l app.kubernetes.io/name=rabbitmq -n openstack --timeout=300s
oc wait --for=condition=Ready pod -l app=memcached -n openstack --timeout=300s
```

**After (with this enhancement):**
```bash
# Single condition check - controller validates all components
oc wait --for=condition=InfrastructureReady openstackcontrolplane/openstack -n openstack --timeout=600s
```

**Result:** Simpler, more reliable, and the controller guarantees all infrastructure is truly ready.

#### Status Conditions

New condition type added:

```go
const (
    // InfrastructureReadyCondition indicates database infrastructure is ready
    InfrastructureReadyCondition condition.Type = "InfrastructureReady"
)
```

**Status when infrastructure is not ready yet (waiting):**

```yaml
status:
  conditions:
  - type: InfrastructureReady
    status: "False"
    lastTransitionTime: "2026-01-23T10:00:00Z"
    message: "Waiting for infrastructure components to be ready: Galera, OVN-NB"
    reason: InfrastructureNotReady
  - type: Ready
    status: "False"
    lastTransitionTime: "2026-01-23T10:00:00Z"
    message: "Deployment paused at infrastructure-only stage, waiting for infrastructure"
    reason: DeploymentPaused
```

**Status when infrastructure is ready (paused for restore):**

```yaml
status:
  conditions:
  - type: InfrastructureReady
    status: "True"
    lastTransitionTime: "2026-01-23T10:05:00Z"
    message: "Infrastructure (Galera, OVN, RabbitMQ, Memcached) is ready for database restore. Remove annotation core.openstack.org/deployment-stage to resume deployment."
    reason: Ready
  - type: Ready
    status: "False"
    lastTransitionTime: "2026-01-23T10:00:00Z"
    message: "Deployment paused at infrastructure-only stage for restore"
    reason: DeploymentPaused
```

#### Infrastructure Scope

Services ready when paused at line ~337 (infrastructure-only stage):

**Infrastructure services (already reconciled):**
- **MariaDB (Galera)** - OpenStack service databases - **needs restore**
- **OVN Northbound Database** - Network logical topology - **needs restore**
- **OVN Southbound Database** - Network physical bindings - **needs restore**
- **RabbitMQ** - Message queue - **needs manual user restoration**
- **Memcached** - Caching layer (ephemeral, no restore needed)

**What gets restored at this pause point:**
1. MariaDB database contents (via `./restore-mariadb.sh`)
2. OVN NB/SB database contents (via `./restore-ovn.sh`)
3. RabbitMQ user credentials (manual restoration for EDPM compatibility)

**Services NOT yet created (paused):**
- Keystone
- Nova (API, Scheduler, Conductor)
- Neutron
- Glance
- Cinder
- Placement
- Heat
- All other OpenStack services

**Memcached note:** It's fine that Memcached is already running - it's ephemeral cache and doesn't need restore.

### Enhanced Restore Workflow

With this enhancement, the restore procedure becomes:

```bash
#!/bin/bash
# Enhanced restore workflow with staged deployment

NAMESPACE="openstack"
BACKUP_DIR="./backups"

echo "=========================================="
echo "OpenStack Control Plane Restore"
echo "Enhanced Workflow with Staged Deployment"
echo "=========================================="
echo ""

# Step 1: Restore control plane in infrastructure-only mode
echo "Step 1: Restoring control plane (infrastructure-only mode)..."
echo ""

# Add staging annotation to backup before restore
# (or modify restore script to add annotation)
./restore-openstack-ctlplane.sh --stage infrastructure-only \
    ${BACKUP_DIR}/openstack-ctlplane-backup-*.tar.gz

echo "✓ Control plane restored in infrastructure-only mode"
echo ""

# Step 2: Wait for infrastructure to be ready
echo "Step 2: Waiting for infrastructure to be ready..."
echo ""

# Simple! Just wait for InfrastructureReady condition
# This automatically validates: Galera, OVN NB, OVN SB, RabbitMQ, Memcached
oc wait --for=condition=InfrastructureReady openstackcontrolplane/openstack -n ${NAMESPACE} --timeout=600s

echo "✓ Infrastructure ready (all components validated by controller)"
echo ""

# Step 3: Restore database contents
echo "Step 3: Restoring database contents..."
echo ""

# Restore MariaDB
echo "Restoring MariaDB databases..."
./restore-mariadb.sh ${BACKUP_DIR}/mariadb/mariadb-backup-*.sql.gz
echo "✓ MariaDB restored"
echo ""

# Restore OVN databases
echo "Restoring OVN databases..."
./restore-ovn.sh \
    ${BACKUP_DIR}/ovn/ovn-nb-backup-*.db \
    ${BACKUP_DIR}/ovn/ovn-sb-backup-*.db
echo "✓ OVN databases restored"
echo ""

# Step 4: Restore RabbitMQ users (if needed for EDPM)
echo "Step 4: Restoring RabbitMQ users..."
# (Manual restoration as documented)
echo "⚠️  Manual RabbitMQ user restoration required for EDPM nodes"
echo "See restore-openstack-ctlplane.sh Step 6 for details"
echo ""

# Step 5: Resume full deployment
echo "Step 5: Resuming full control plane deployment..."
echo ""

oc annotate openstackcontrolplane openstack -n ${NAMESPACE} \
    core.openstack.org/deployment-stage-

echo "✓ Deployment stage annotation removed"
echo ""

# Step 6: Wait for all services
echo "Step 6: Waiting for all OpenStack services to be ready..."
echo ""

oc wait --for=condition=Ready openstackcontrolplane/openstack \
    -n ${NAMESPACE} --timeout=1200s

echo ""
echo "=========================================="
echo "Restore Complete"
echo "=========================================="
echo ""
echo "OpenStack services started with restored databases"
echo "No service restarts required"
echo ""
```

### Comparison: Current vs Enhanced Workflow

#### Current Workflow

```
1. Restore CR
2. ALL pods created (databases + services)
   ├─ Services start with EMPTY databases
   ├─ Services may error or create schemas
   └─ Race condition: services vs database readiness
3. Wait for MULTIPLE components individually:
   ├─ Wait for Galera pods
   ├─ Wait for OVN NB pod
   ├─ Wait for OVN SB pod
   ├─ Wait for RabbitMQ pods
   └─ Wait for Memcached pods
4. Restore databases into RUNNING pods
5. Services need to restart/reconnect
6. Potential schema conflicts
```

**Timeline:**
```
T+0:   Restore CR
T+1:   MariaDB pod starting, Keystone pod starting (empty DB!)
T+2:   Keystone errors (no DB schema)
T+5:   MariaDB ready
T+10:  Restore databases
T+15:  Restart Keystone (sees restored data)
T+20:  System stable
```

#### Enhanced Workflow

```
1. Restore CR (infrastructure-only mode)
2. ONLY infrastructure pods created
   └─ No services running yet
3. Wait for SINGLE condition: InfrastructureReady ⭐
   └─ Controller validates all components automatically
4. Restore databases into RUNNING pods
5. Resume deployment (remove annotation)
6. Services start with RESTORED databases
   └─ No restarts needed
```

**Timeline:**
```
T+0:   Restore CR (infrastructure-only)
T+1:   MariaDB pod starting, NO other services
T+5:   InfrastructureReady=True (all validated!)
T+10:  Restore databases
T+11:  Resume deployment (remove annotation)
T+12:  Keystone pod starting (with RESTORED DB!)
T+15:  System stable
```

**Benefits:**
- ✅ Faster restore (no restarts)
- ✅ No schema conflicts
- ✅ No service errors
- ✅ Cleaner operational workflow
- ✅ More predictable timing
- ✅ **Simpler restore script**: Single condition check instead of multiple component checks

## User Experience

### Restore Script Changes

The `restore-openstack-ctlplane.sh` script would be enhanced:

```bash
#!/bin/bash

# ... existing code ...

STAGE="${STAGE:-complete}"  # New option

# Support --stage flag
while [[ $# -gt 0 ]]; do
    case $1 in
        --stage)
            STAGE="$2"
            shift 2
            ;;
        --stage=*)
            STAGE="${1#*=}"
            shift
            ;;
        # ... other options ...
    esac
done

# ... restore resources ...

# Apply staging annotation if requested
if [ "${STAGE}" = "infrastructure-only" ]; then
    echo "Setting deployment stage to: infrastructure-only"
    oc annotate openstackcontrolplane openstack -n ${NAMESPACE} \
        core.openstack.org/deployment-stage=infrastructure-only \
        --overwrite
fi
```

### Resume Script

Create a simple resume script:

```bash
#!/bin/bash
# resume-deployment.sh

NAMESPACE="${NAMESPACE:-openstack}"

echo "Resuming OpenStack control plane deployment..."
echo ""

# Remove staging annotation
oc annotate openstackcontrolplane openstack -n ${NAMESPACE} \
    core.openstack.org/deployment-stage-

echo "✓ Deployment resumed (annotation removed)"
echo ""
echo "Waiting for all services to be ready..."
oc wait --for=condition=Ready openstackcontrolplane/openstack \
    -n ${NAMESPACE} --timeout=1200s

echo ""
echo "✓ All services ready"
```

### Documentation Updates

Update all restore documentation to mention staged deployment:

**backup-restore-ctlplane.md:**
```markdown
## Restore Procedure

### Option 1: Staged Restore (Recommended for Production)

Use staged deployment to restore databases before services start:

1. Restore control plane in infrastructure-only mode
2. Restore database contents
3. Resume full deployment

See [Enhanced Restore Workflow](#enhanced-restore-workflow)

### Option 2: Standard Restore

Restore all services at once and restore databases afterward:

1. Restore control plane
2. Wait for database pods
3. Restore database contents
4. Restart services

See [Current Restore Workflow](#current-restore-workflow)
```

## Implementation Phases

### Phase 1: Basic Annotation Support (Minimum Viable Product)

**Scope:**
- Add annotation check at line ~337 in controller
- Add early return if `infrastructure-only` stage
- Set status conditions
- Update restore scripts with `--stage` flag

**Code changes required:**
- ~50 lines in `openstackcontrolplane_controller.go` (status updated by existing defer function)
  - Annotation check
  - Infrastructure readiness validation (Galera, OVN NB/SB, RabbitMQ, Memcached)
  - Status condition management
- Update to `restore-openstack-ctlplane.sh` to set annotation
- Documentation updates

**Deliverables:**
- Controller change (if statement + readiness checks + status conditions)
- Unit test for annotation handling and readiness validation
- E2E test for staged deployment
- Documentation updates

**Estimated Effort:** 1-2 days of development + testing

**Why still simple:**
The controller already has the perfect pause point at line ~337. All infrastructure is reconciled, services haven't started yet. Check the annotation, validate infrastructure is ready, set status conditions, and return early. Status update handled by existing defer function.

### Phase 2: Enhanced Control and Validation (Optional)

**Scope:**
- Validate stage transitions
- Enhanced status reporting with more details
- Metrics for staged deployments
- Timeout handling (auto-resume after X hours?)

**Deliverables:**
- Stage transition validation
- Enhanced status conditions
- Prometheus metrics
- Operator documentation

**Estimated Effort:** 2-3 days

### Phase 3: Additional Stages (Future Enhancement - Optional)

**Scope:**
- Support for more granular stages (if needed)
- Multiple pause points throughout reconciliation
- Dependency graph validation

**Note:** This is probably NOT needed. The single pause point at line ~337 covers the restore use case perfectly.

**Estimated Effort:** Only if needed in the future

## Testing Strategy

### Unit Tests

```go
func TestStagedDeployment(t *testing.T) {
    // Test infrastructure-only annotation
    instance := &corev1beta1.OpenStackControlPlane{
        ObjectMeta: metav1.ObjectMeta{
            Annotations: map[string]string{
                "core.openstack.org/deployment-stage": "infrastructure-only",
            },
        },
    }

    // Verify only database resources are created
    // Verify InfrastructureReady condition is set
    // Verify other services are NOT created
}

func TestStageTransition(t *testing.T) {
    // Start with infrastructure-only
    // Verify databases created
    // Change to complete
    // Verify all services created
}
```

### E2E Tests

```yaml
# E2E test scenario
apiVersion: test.openstack.org/v1
kind: OpenStackControlPlaneTest
metadata:
  name: staged-deployment-test
spec:
  steps:
  - name: deploy-infrastructure-only
    apply:
      apiVersion: core.openstack.org/v1beta1
      kind: OpenStackControlPlane
      metadata:
        annotations:
          core.openstack.org/deployment-stage: infrastructure-only
      spec:
        # ... minimal spec
    wait:
      condition: InfrastructureReady
      timeout: 10m

  - name: verify-only-databases
    verify:
      - mariadb pods exist
      - ovn database pods exist
      - keystone pods do NOT exist
      - nova pods do NOT exist

  - name: resume-deployment
    patch:
      apiVersion: core.openstack.org/v1beta1
      kind: OpenStackControlPlane
      metadata:
        annotations:
          core.openstack.org/deployment-stage: complete
    wait:
      condition: Ready
      timeout: 30m

  - name: verify-all-services
    verify:
      - keystone pods exist
      - nova pods exist
      - all services ready
```

### Integration Tests

Test restore workflow:

```bash
#!/bin/bash
# test-staged-restore.sh

# 1. Deploy full control plane
# 2. Create test data in databases
# 3. Backup control plane
# 4. Delete control plane
# 5. Restore with infrastructure-only stage
# 6. Verify only databases running
# 7. Restore database contents
# 8. Verify data present in databases
# 9. Resume to complete stage
# 10. Verify all services running with restored data
# 11. Verify no service restarts occurred
```

## Backwards Compatibility

### No Breaking Changes

- **Default behavior unchanged**: Without annotation, deployment works as it does today
- **Existing deployments unaffected**: Annotation only checked for new reconciliations
- **Opt-in feature**: Users must explicitly add annotation to use staged deployment

### Migration Path

Users can adopt staged deployment incrementally:

1. **Current users**: Continue with existing restore procedures
2. **New adopters**: Use staged deployment from day one
3. **Gradual migration**: Test staged deployment in dev/test before production

## Security Considerations

### Potential Risks

1. **Database exposure**: Databases running without application layer might be exposed
   - **Mitigation**: Network policies remain in place, only in-cluster access

2. **Incomplete deployment state**: Control plane in infrastructure-only state might be confusing
   - **Mitigation**: Clear status conditions, documentation, automated timeout

### Access Control

No new RBAC permissions required:
- Same permissions needed to create/update OpenStackControlPlane
- Database restore requires exec permissions (already needed)

## Alternatives Considered

### Alternative 1: Separate Restore CR

**Approach**: Create a dedicated `OpenStackControlPlaneRestore` CR

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlaneRestore
metadata:
  name: restore-20260123
spec:
  controlPlaneRef:
    name: openstack
  stage: databases
  backupSource:
    # ... backup location
```

**Pros:**
- Clear separation of concerns
- Dedicated restore workflow
- Could automate entire restore process

**Cons:**
- New CRD to maintain
- More complex API
- Duplicates backup/restore tooling

**Decision**: Deferred - could be Phase 4 enhancement, but annotation approach is simpler for MVP

## Future Enhancements

### Automated Restore Orchestration

Controller could orchestrate full restore:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlaneRestore
metadata:
  name: restore-20260123
spec:
  controlPlaneRef:
    name: openstack
  backupSource:
    mariadb:
      pvc: backup-pvc
      path: /backups/mariadb-20260123.sql.gz
    ovn:
      pvc: backup-pvc
      nbPath: /backups/ovn-nb-20260123.db
      sbPath: /backups/ovn-sb-20260123.db
  autoResume: true  # Automatically resume after restore
```

Controller would:
1. Create control plane in infrastructure-only mode
2. Wait for databases
3. Restore database contents from specified sources
4. Resume deployment automatically
5. Set status with restore progress

### Backup CR Integration

Integrate with backup CRs for complete automation:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlaneBackup
metadata:
  name: daily-backup
spec:
  controlPlaneRef:
    name: openstack
  schedule: "0 2 * * *"  # 2 AM daily
  retention:
    daily: 7
    weekly: 4
  destination:
    pvc: backup-pvc
```

### Multi-Stage Deployment

Support more granular stages:

```yaml
annotations:
  core.openstack.org/deployment-stage: "core-services"
```

**Stages:**
- `infrastructure` - Networks, storage
- `databases` - MariaDB, OVN, RabbitMQ
- `identity` - Keystone only
- `core-services` - Keystone, Glance, Placement
- `compute` - Nova
- `network` - Neutron
- `complete` - Everything

## Success Criteria

### Functional Requirements

- ✅ Controller respects deployment-stage annotation
- ✅ infrastructure-only stage creates only database infrastructure
- ✅ InfrastructureReady condition set when databases operational
- ✅ Transition to complete stage creates remaining services
- ✅ No service restarts required after database restore
- ✅ Restore scripts updated to support staged deployment

### Performance Requirements

- ✅ Stage transition completes within 5 minutes
- ✅ No performance degradation in normal (non-staged) deployments
- ✅ Database restore time unchanged

### Documentation Requirements

- ✅ Enhancement document (this document)
- ✅ User guide for staged restore
- ✅ Operator development guide
- ✅ E2E test documentation

## References

### Related Documentation

- [Kubernetes API Conventions - Annotations](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#annotations)
- [Operator Best Practices](https://sdk.operatorframework.io/docs/best-practices/)
- [OpenStack Operator Architecture](https://github.com/openstack-k8s-operators/docs)

### Similar Patterns in Other Operators

- **Istio**: `istioctl install --set profile=minimal` for staged installation
- **ArgoCD**: Sync waves for ordered deployment
- **Velero**: Restore hooks for pre/post restore operations
- **Strimzi Kafka**: Phased rolling updates

## Appendix A: API Examples

### Complete Staged Restore Example

```yaml
---
# Step 1: OpenStackControlPlane with infrastructure-only annotation
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
  namespace: openstack
  annotations:
    # Start in infrastructure-only mode
    core.openstack.org/deployment-stage: "infrastructure-only"
spec:
  storageClass: local-storage
  secret: openstack-secret
  galera:
    enabled: true
    templates:
      galera:
        replicas: 3
        storageRequest: 10Gi
  ovn:
    enabled: true
    template:
      ovnDBCluster:
        ovndbcluster-nb:
          replicas: 3
        ovndbcluster-sb:
          replicas: 3
  # ... rest of spec

---
# After database restore, resume deployment
# Update annotation:
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
  namespace: openstack
  annotations:
    # Resume full deployment
    core.openstack.org/deployment-stage: "complete"
spec:
  # ... same spec as above
```

### Status During Staged Deployment

```yaml
# Status while in infrastructure-only stage
status:
  conditions:
  - type: InfrastructureReady
    status: "True"
    lastTransitionTime: "2026-01-23T10:00:00Z"
    message: "Database infrastructure is ready for restore"
    reason: Ready
    observedGeneration: 1
  - type: Ready
    status: "False"
    lastTransitionTime: "2026-01-23T10:00:00Z"
    message: "Deployment paused at infrastructure-only stage"
    reason: DeploymentStaged
    observedGeneration: 1
  observedGeneration: 1

# Status after transitioning to complete
status:
  conditions:
  - type: InfrastructureReady
    status: "True"
    lastTransitionTime: "2026-01-23T10:00:00Z"
    message: "Database infrastructure is ready"
    reason: Ready
    observedGeneration: 2
  - type: KeystoneReady
    status: "True"
    lastTransitionTime: "2026-01-23T10:05:00Z"
    message: "Keystone is ready"
    reason: Ready
    observedGeneration: 2
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-01-23T10:15:00Z"
    message: "OpenStack control plane is ready"
    reason: Ready
    observedGeneration: 2
  observedGeneration: 2
```

## Appendix B: Actual Controller Code Change

**Location:** `internal/controller/core/openstackcontrolplane_controller.go` around line 337

**Current code (simplified):**
```go
func (r *OpenStackControlPlaneReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // ... fetch instance, initial setup ...

    // Reconcile OVN
    // ...

    // Reconcile RabbitMQ
    // ...

    // Reconcile Galera
    // ...

    // Reconcile Memcached
    // ...

    // Line ~337 - Continue to reconcile services
    // Reconcile Keystone
    // Reconcile Nova
    // etc.
}
```

**New code (with pause point):**
```go
func (r *OpenStackControlPlaneReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // ... fetch instance, initial setup ...

    // Reconcile OVN
    // ...

    // Reconcile RabbitMQ
    // ...

    // Reconcile Galera
    // ...

    // Reconcile Memcached
    // ...

    // **NEW CODE - Check if we should pause for infrastructure restore**
    if instance.Annotations != nil && instance.Annotations["core.openstack.org/deployment-stage"] == "infrastructure-only" {
        // Check if all infrastructure components are ready
        infraReady := true
        var notReadyComponents []string

        // Check Galera
        if instance.Spec.Galera.Enabled {
            galera := &mariadbv1.Galera{}
            if err := r.Get(ctx, types.NamespacedName{Name: "galera", Namespace: instance.Namespace}, galera); err != nil || !galera.Status.Conditions.IsTrue(condition.ReadyCondition) {
                infraReady = false
                notReadyComponents = append(notReadyComponents, "Galera")
            }
        }

        // Check OVN databases
        if instance.Spec.Ovn.Enabled {
            ovnNB := &ovnv1.OVNDBCluster{}
            if err := r.Get(ctx, types.NamespacedName{Name: "ovndbcluster-nb", Namespace: instance.Namespace}, ovnNB); err != nil || !ovnNB.Status.Conditions.IsTrue(condition.ReadyCondition) {
                infraReady = false
                notReadyComponents = append(notReadyComponents, "OVN-NB")
            }

            ovnSB := &ovnv1.OVNDBCluster{}
            if err := r.Get(ctx, types.NamespacedName{Name: "ovndbcluster-sb", Namespace: instance.Namespace}, ovnSB); err != nil || !ovnSB.Status.Conditions.IsTrue(condition.ReadyCondition) {
                infraReady = false
                notReadyComponents = append(notReadyComponents, "OVN-SB")
            }
        }

        // Check RabbitMQ
        // TODO: Add RabbitMQ readiness check

        // Check Memcached
        // TODO: Add Memcached readiness check

        if infraReady {
            // All infrastructure ready - set condition and pause
            instance.Status.Conditions.Set(condition.TrueCondition(
                corev1beta1.OpenStackControlPlaneInfrastructureReadyCondition,
                condition.ReadyMessage,
                "Infrastructure (Galera, OVN, RabbitMQ, Memcached) is ready for database restore. "+
                "Remove annotation core.openstack.org/deployment-stage to resume deployment.",
            ))

            instance.Status.Conditions.Set(condition.FalseCondition(
                condition.ReadyCondition,
                "DeploymentPaused",
                condition.SeverityInfo,
                "Deployment paused at infrastructure-only stage for restore",
            ))
            r.Log.Info("Deployment paused at infrastructure-only stage - infrastructure ready for restore")
        } else {
            // Infrastructure not ready yet - set waiting condition
            instance.Status.Conditions.Set(condition.FalseCondition(
                corev1beta1.OpenStackControlPlaneInfrastructureReadyCondition,
                "InfrastructureNotReady",
                condition.SeverityInfo,
                fmt.Sprintf("Waiting for infrastructure components to be ready: %s", strings.Join(notReadyComponents, ", ")),
            ))

            instance.Status.Conditions.Set(condition.FalseCondition(
                condition.ReadyCondition,
                "DeploymentPaused",
                condition.SeverityInfo,
                "Deployment paused at infrastructure-only stage, waiting for infrastructure",
            ))
            r.Log.Info("Deployment paused at infrastructure-only stage - waiting for infrastructure", "notReady", notReadyComponents)
        }

        // Status will be updated by defer function
        // Pause here - don't continue to service reconciliation
        return ctrl.Result{}, nil
    }
    // **END NEW CODE**

    // Line ~337 - Continue to reconcile services (only if not paused)
    // Reconcile Keystone
    // Reconcile Nova
    // etc.
}
```

**That's the entire change!** ~50 lines of code added at the perfect location:
- Annotation check
- Infrastructure readiness validation
- Status condition management
- No explicit status update needed - handled by defer function

## Appendix C: Metrics

Add Prometheus metrics for observability:

```go
var (
    controlPlaneDeploymentStage = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "openstack_controlplane_deployment_stage",
            Help: "Current deployment stage (0=complete, 1=infrastructure-only)",
        },
        []string{"namespace", "name", "stage"},
    )

    controlPlaneInfrastructureReadyTime = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "openstack_controlplane_databases_ready_duration_seconds",
            Help:    "Time taken for databases to become ready",
            Buckets: prometheus.ExponentialBuckets(10, 2, 10), // 10s to ~2.5h
        },
        []string{"namespace", "name"},
    )
)

func init() {
    metrics.Registry.MustRegister(
        controlPlaneDeploymentStage,
        controlPlaneInfrastructureReadyTime,
    )
}
```

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-23 | OpenStack Operator Team | Initial proposal |
