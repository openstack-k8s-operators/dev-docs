# Application Credentials

## Purpose

Application Credentials (ACs) provide an alternative authentication mechanism for OpenStack services. Instead of using the service user's password from `osp-secret`, services authenticate using a Keystone Application Credential: a time-restricted, project-scoped credential with its own ID and secret.

This serves two goals:

1. **Scoped, rotatable credentials**: ACs have a defined expiration and can be automatically rotated without human intervention. Each AC can be restricted to specific roles and access rules.

2. **Password rotation enabler**: Once AC is enabled for a service, the service authenticates via `v3applicationcredential` auth type and no longer uses the password from `osp-secret` at runtime. This means the password in `osp-secret` can be rotated without disrupting running services. The keystone-operator AC controller still reads the password from `osp-secret` when creating or rotating ACs (to authenticate as the service user in Keystone), but it fetches it on-demand at reconcile time, so a rotated password is picked up automatically.

## Architecture Overview

Three layers of operators are involved:

```text
┌─────────────────────────────────────────────────────┐
│  openstack-operator                                 │
│  - Owns KeystoneApplicationCredential CRs           │
│  - Creates/patches AC CRs                           │
│  - Reads Status.SecretName from AC CR               │
│  - Sets ApplicationCredentialSecret on service CRs  │
└──────────────────────────┬──────────────────────────┘
                           │ creates AC CR
                           ▼
┌─────────────────────────────────────────────────────┐
│  keystone-operator                                  │
│  - Reconciles KeystoneApplicationCredential CRs     │
│  - Creates AC in Keystone (API call)                │
│  - Creates K8s Secret with AC_ID and AC_SECRET      │
│  - Manages rotation                                 │
│  - Sets Status on AC CRs                            │
└──────────────────────────┬──────────────────────────┘
                           │ AC secret name flows to service CR
                           ▼
┌─────────────────────────────────────────────────────┐
│  service operators                                  │
│  - Watch the named AC secret via field indexer      │
│  - Read AC_ID and AC_SECRET from the secret         │
│  - Generate service config with auth_type =         │
│    v3applicationcredential                          │
└─────────────────────────────────────────────────────┘
```

## Configuration

### Global AC settings

Set on the `OpenStackControlPlane` CR under `spec.applicationCredential`:

```yaml
spec:
  applicationCredential:
    enabled: true
    expirationDays: 730      # default: 730 (min 2)
    gracePeriodDays: 364     # default: 364 (min 1, must be < expirationDays)
    roles:                   # default: [admin, service]
      - admin
      - service
    unrestricted: false      # default: false
```

### Per-service overrides

Each service has an `applicationCredential` section that can override global defaults. In order to enable AC for a service, both global and service-specific fields must be enabled. Omitted fields inherit from the global config:

```yaml
spec:
  barbican:
    applicationCredential:
      enabled: true
      expirationDays: 365    # override: shorter expiry for barbican
      roles:                 # override: service role only
        - service
```

### Special cases

- **Telemetry** has three independent sub-services (Aodh, Ceilometer, CloudKitty), each with its own AC CR and config section (`applicationCredentialAodh`, `applicationCredentialCeilometer`, `applicationCredentialCloudKitty`).
- **Ironic** creates two AC CRs: one for `ironic` and one for `ironic-inspector`.
- **Glance** creates a single AC CR shared across all GlanceAPI instances.
- **EDPM AC consumers** are **Nova** and **Ceilometer**. These services have an additional operational gap because the credentials are deployed to EDPM nodes as static configuration and are not picked up automatically after rotation, so the manual redeployment during grace period window is neccesary.

## Workflow

### AC Creation

1. `openstack-operator` reconciles the `OpenStackControlPlane` CR.
2. For each service with AC enabled, it calls `EnsureApplicationCredentialForService`.
3. If the service is ready and no AC CR exists, `reconcileApplicationCredential` creates a `KeystoneApplicationCredential` CR (for example, `ac-barbican`) with the merged config. The CR is owned by the `OpenStackControlPlane`.
4. `keystone-operator` reconciles the AC CR:
   - Authenticates as the service user using the password from `osp-secret`.
   - Creates an Application Credential in Keystone with the specified roles, expiration, and access rules.
   - Creates a K8s Secret named `<ac-cr-name>-secret` (for example, `ac-barbican-secret`) containing `AC_ID` and `AC_SECRET`.
   - Sets `Status.SecretName`, `Status.ACID`, and `Status.ExpiresAt` on the AC CR.
   - Adds a protection finalizer (`openstack.org/ac-secret-protection`) to the Secret.
5. The `openstack-operator` `Owns` the AC CR, so when `keystone-operator` updates its status (sets Ready, populates `Status.SecretName`), controller-runtime triggers a reconcile of the owning `OpenStackControlPlane`. The `openstack-operator` sees the AC CR is ready, reads `Status.SecretName`, and sets it on the service CR (for example, `barbican.spec.auth.applicationCredentialSecret`).
6. The service operator (for example, `barbican-operator`) watches the named Secret via a field indexer, reads `AC_ID` and `AC_SECRET`, and generates config with `auth_type = v3applicationcredential`.

### AC Rotation

Rotation is triggered by the `keystone-operator` AC controller during reconcile.

#### Rotation triggers

The controller first evaluates `needsRotation()`, which returns true in these cases:

1. **No AC exists yet** — `Status.ACID` is empty (initial creation).
2. **Security-critical fields changed** — `SecurityHash` (computed from `roles`, `accessRules`, `unrestricted`) differs from the stored hash. This triggers immediate rotation.
3. **Grace period reached** — `time.Now()` is past `ExpiresAt - GracePeriodDays`.

In addition, `reconcileNormal()` performs a Secret existence check:
4. **AC Secret missing** — if `Status.SecretName` is set but the referenced Secret does not exist, the controller forces rotation.

#### Rotation behavior

On rotation:

- A new AC is created in Keystone with a fresh random suffix in the name.
- The existing K8s Secret name stays the same (`<ac-cr-name>-secret`) and is updated with the new `AC_ID` and `AC_SECRET`.
- The old AC in Keystone is **not** revoked, it expires naturally.
- The Secret content change triggers the service operator to reconcile and regenerate config.
- When rotating an existing AC (not initial create), the controller emits a Kubernetes event on the `KeystoneApplicationCredential` CR: `ApplicationCredentialRotated`.

### Manual Rotation and Test Triggers

For engineering and testing purposes, the following approaches can be used to trigger rotation or recreation:

#### Supported immediate rotation path

Change one of the security-critical fields in the AppCred spec:

- `roles`
- `accessRules`
- `unrestricted`

This changes the computed `securityHash` and causes a normal immediate rotation on the next reconcile.

#### Trigger by patching `status.expiresAt`

Patching `status.expiresAt` to a timestamp in the past can be used to make the controller consider the credential inside the grace window and perform a normal rotation path on the next reconcile.

This differs from deleting the Secret because:

- the existing Secret object remains present,
- service operators do not transiently reconcile into `ErrACSecretNotFound`,
- the rotation proceeds through the standard in-place Secret update path.

#### Recovery path by deleting the Secret

The referenced AC Secret is protected by the `openstack.org/ac-secret-protection` finalizer, so it is not normally removable with a simple delete.

If the Secret does become absent (for example, after intentionally removing/bypassing the protection finalizer), `keystone-operator` detects that `Status.SecretName` points to a missing Secret and forces rotation/recreation.

Because this introduces a window where service operators may reconcile into `ErrACSecretNotFound`, this is better treated as an engineering recovery path than as a normal rotation trigger.

#### Recreate path by deleting the AppCred CR

Deleting the `KeystoneApplicationCredential` CR is a different path.

During delete, `keystone-operator`:
- removes the Secret protection finalizer,
- removes its own AppCred CR finalizer,
- does not revoke the Keystone-side application credential.

If the service remains enabled, `openstack-operator` recreates the AppCred CR on a later reconcile.

This is closer to delete-and-recreate than to a normal in-place rotation.

### AC Cleanup

AC CRs are deleted in two scenarios:

1. **AC disabled** — When `applicationCredential.enabled` is set to `false` (globally or per-service), `EnsureApplicationCredentialForService` detects `!isACEnabled` and deletes the AC CR if it exists.
2. **Service disabled** — When a service is disabled (for example, `spec.barbican.enabled: false`), `CleanupApplicationCredentialForService` unconditionally deletes the AC CR regardless of the AC enabled flag.

When the AC CR is deleted, the `keystone-operator`:

- Removes the protection finalizer from the Secret.
- Removes its own finalizer from the CR.
- Does **not** revoke the AC in Keystone; it expires naturally.

Because the Secret is owned by the AC CR (owner reference), it is garbage-collected after the AC CR is deleted and the protection finalizer is removed.

> **Note:** If immediate Keystone-side cleanup is needed (for example, a suspected credential leak), the AC can be manually deleted from Keystone:
>
> `openstack application credential delete <AC_ID>`
>
> Be careful: deleting an AC that is still in use, especially by EDPM, will cause authentication failures.

The service falls back to password auth on the next reconcile when `ApplicationCredentialSecret` is cleared.

### How Service Operators Consume ACs

Service operators follow a consistent pattern:

- **Field indexer** on `.spec.auth.applicationCredentialSecret` allows lookup of which service CRs reference a given Secret.
- **Secret watch** with `ResourceVersionChangedPredicate` triggers reconciliation when the AC Secret content changes.
- **Config generation** reads `AC_ID` and `AC_SECRET` from the Secret and renders the service config with `auth_type = v3applicationcredential`.
- **Two distinct states to distinguish**:
  - **AC not configured** (`ApplicationCredentialSecret` is empty on the service CR) — the config template renders with `auth_type = password`. This is the normal state when AC is disabled or was never enabled.
  - **AC Secret missing** (`ApplicationCredentialSecret` is set but the referenced Secret doesn't exist) — the service operator returns `ErrACSecretNotFound` and the reconcile fails. Existing pods continue running with old config until they are restarted.

## Control Plane Operations and Observability

### List Application Credentials

List all AppCred CRs in the control plane namespace:

```bash
oc get appcred -n openstack
```

Example output:

```text
NAME                  ACID                               SECRETNAME                   LASTROTATED            ROTATIONELIGIBLE       STATUS   MESSAGE
ac-barbican           d38dc4310fbf4601bbe9f4234eb24114   ac-barbican-secret           2026-03-12T08:23:58Z   2026-03-15T08:23:58Z   True     Setup complete
ac-ceilometer         0e0e7c8f243c4ce8913fd709d44c0104   ac-ceilometer-secret                                2028-02-29T13:35:06Z   True     Setup complete
ac-cinder             b8c7fb9d3abc4ce18727a56f870c9a18   ac-cinder-secret                                    2027-03-03T13:04:41Z   True     Setup complete
ac-glance             da20e5b59d0f4227938046c60857cb62   ac-glance-secret             2026-03-12T08:31:42Z   2027-03-13T08:31:42Z   True     Setup complete
ac-ironic             ec6bbfb36b8041cbad96f4360a84cd71   ac-ironic-secret                                    2027-03-03T13:04:42Z   True     Setup complete
ac-ironic-inspector   81aebe21c2b94a2b89caec4b1fe652c1   ac-ironic-inspector-secret                          2027-03-03T13:04:45Z   True     Setup complete
ac-octavia            0dd0b1b7247743b980d909334a4ea8ad   ac-octavia-secret                                   2027-03-07T09:39:19Z   True     Setup complete
```

The main columns are:

- `NAME` — AppCred CR name
- `ACID` — Keystone Application Credential ID
- `SECRETNAME` — Kubernetes Secret currently holding `AC_ID` and `AC_SECRET`
- `LASTROTATED` — last successful rotation time
- `ROTATIONELIGIBLE` — time when automatic rotation becomes eligible
- `STATUS` / `MESSAGE` — summarized readiness state

### Inspect a single AppCred CR

```bash
oc describe appcred -n openstack ac-barbican
oc get appcred -n openstack ac-barbican -o yaml
```

Example CR:

```yaml
apiVersion: keystone.openstack.org/v1beta1
kind: KeystoneApplicationCredential
metadata:
  name: ac-barbican
  namespace: openstack
  finalizers:
    - openstack.org/applicationcredential
  ownerReferences:
    - apiVersion: core.openstack.org/v1beta1
      kind: OpenStackControlPlane
      name: openstack-galera-network-isolation
      controller: true
      blockOwnerDeletion: true
spec:
  expirationDays: 5
  gracePeriodDays: 2
  passwordSelector: BarbicanPassword
  roles:
    - service
  secret: osp-secret
  unrestricted: false
  userName: barbican
status:
  acID: d38dc4310fbf4601bbe9f4234eb24114
  secretName: ac-barbican-secret
  createdAt: "2026-03-12T08:23:58Z"
  expiresAt: "2026-03-17T08:23:58Z"
  lastRotated: "2026-03-12T08:23:58Z"
  rotationEligibleAt: "2026-03-15T08:23:58Z"
  observedGeneration: 1
  conditions:
    - type: Ready
      status: "True"
      reason: Ready
      message: Setup complete
    - type: KeystoneAPIReady
      status: "True"
      reason: Ready
      message: KeystoneAPI ready
    - type: KeystoneApplicationCredentialReady
      status: "True"
      reason: Ready
      message: ApplicationCredential ready
```

### Rotation events

On rotation, `keystone-operator` emits a Kubernetes event on the AppCred CR with reason `ApplicationCredentialRotated`.

Examples:

```bash
oc get events -n openstack --sort-by=.lastTimestamp | grep ApplicationCredentialRotated
```

Output:
```bash
8s  Normal  ApplicationCredentialRotated   keystoneapplicationcredential/ac-barbican    ApplicationCredential 'ac-barbican' (user: barbican) rotated - EDPM nodes may need credential updates. Previous expiration: 2001-05-19T00:00:00Z, New expiration: 2026-03-18T08:24:18Z
```

For EDPM consumers such as **Nova** and **Ceilometer**, these events are particularly important because the rotated credential still needs to be propagated to the data plane through redeployment.

## Conditions and States

The AppCred CR does not expose a dedicated phase/status enum such as `Creating`, `Rotating`, or `Failed`. Instead, it exposes conditions and timestamps.

The main conditions are:

- `Ready`
- `KeystoneAPIReady`
- `KeystoneApplicationCredentialReady`

Typical interpretations are:

- **Ready / healthy**
  - `Ready=True`
  - `KeystoneAPIReady=True`
  - `KeystoneApplicationCredentialReady=True`

- **Waiting for Keystone**
  - `KeystoneAPIReady=False`
  - indicates KeystoneAPI is not yet ready or reachable

- **Application credential setup failed**
  - `KeystoneApplicationCredentialReady=False`
  - inspect the condition reason/message and controller logs

- **Rotated**
  - there is no separate `Rotated` condition
  - use `lastRotated`, `rotationEligibleAt`, `acID`, and events such as `ApplicationCredentialRotated`

- **Not reconciled to latest spec yet**
  - compare `metadata.generation` with `status.observedGeneration`


## Current Limitations

1. **Mutable secrets** — Rotation overwrites the existing Secret in place. If propagation fails mid-rotation, the old credential is gone and there is no rollback path.

2. **No service-side finalizers** — Service operators do not add finalizers to the AC Secret. The `keystone-operator` adds its own protection finalizer (`openstack.org/ac-secret-protection`) to prevent accidental deletion, but there is no service-side finalizer to coordinate credential handoff during rotation. On the control plane this is self-healing — if an AC CR or its Secret is deleted, the `openstack-operator` detects the change and recreates it almost immediately. The real gap is that the existing Secret is overwritten in place before the service has confirmed it picked up the new credentials, with no previous version to fall back to.

3. **No Keystone revocation** — Old ACs are not revoked in Keystone after rotation or CR deletion. They persist until they expire, which can be a long time.

4. **EDPM gap** — EDPM nodes have AC credentials deployed as static config files via Ansible. There is no watch or reconcile loop on the node — if the AC rotates, EDPM nodes continue using the old credentials silently until a manual `OpenStackDataPlaneNodeSet` deployment is triggered. If the old AC expires in Keystone before the nodes are updated, services on those nodes will start getting authentication errors. This currently applies to **Nova** and **Ceilometer**.

## Future Improvements

The following changes have been agreed upon for the next iteration:

#### Immutable Secrets with Unique Names

Each rotation will create a new **immutable** K8s Secret with a unique name that includes the first 5 characters of the Keystone AC ID (for example, `ac-barbican-a1b2c-secret`). The old Secret is preserved until all consumers confirm they have switched to the new credential. This enables safe rollback — if propagation fails, the old Secret and credential remain valid and in use.

#### Finalizer Based Lifecycle

When a service operator starts using an AC Secret, it adds a service-specific finalizer to the Secret. The `keystone-operator` only deletes the old Secret and revokes the old AC in Keystone after all service finalizers have been removed, confirming that every consumer has switched to the new credential.

#### Service Labels on AC Secrets

AC Secrets will get a label identifying the owning service (for example, `app-credential-service: barbican`). This allows easy discovery and cleanup of all AC Secrets for a specific service.

#### Keystone AC Revocation

With the finalizer handshake in place, it becomes safe to revoke old ACs in Keystone after all consumers have confirmed the switch. This closes the credential leakage gap where old ACs persist for a long period of time.

#### EDPM Finalizer Coordination

The `OpenStackDataPlaneNodeSet` controller will add a finalizer to the AC Secret it is currently using. The finalizer is removed only after all nodes have been redeployed with the new Secret. This ensures credentials are not revoked while EDPM nodes are still using them. This work depends on the secret-tracking mechanism from PR [#1781](https://github.com/openstack-k8s-operators/openstack-operator/pull/1781).

#### Target Rotation Flow

```text
1. keystone-operator detects rotation needed (grace period, security hash change, or missing Secret)
2. Creates new AC in Keystone and a new immutable Secret with a unique name
3. Updates AC CR Status.SecretName to point to the new Secret
4. openstack-operator watches the AC CR (via Owns), sees the new SecretName, and updates the service CR
5. Service operator watches the named Secret field, picks up the new Secret, and reconfigures
6. Service operator removes its finalizer from the old Secret
7. keystone-operator sees the old Secret has no service finalizers, deletes it, and revokes the old AC
```

For EDPM services, step 6 will require OpenStackDataPlaneNodeSet level tracking of which secret version is deployed on each node. Automatic finalizer removal after all nodes have adopted the new credential depends on the per-node secret tracking work proposed in PR [#1781](https://github.com/openstack-k8s-operators/openstack-operator/pull/1781).