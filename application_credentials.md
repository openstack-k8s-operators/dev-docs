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
│  - Creates/patches AC CRs (incl. EDPM annotation)   │
│  - Reads Status.SecretName from AC CR               │
│  - Sets spec.auth.applicationCredentialSecret on    │
│    service CR templates                             │
└──────────────────────────┬──────────────────────────┘
                           │ creates AC CR
                           ▼
┌─────────────────────────────────────────────────────┐
│  keystone-operator                                  │
│  - Creates/rotates ACs in Keystone                  │
│  - Creates immutable Secrets                        │
│  - Protection finalizer on Secrets                  │
│  - Revokes unused ACs; deletes old Secrets when safe│
│  - Gates cleanup/delete on EDPM NodeSet hash sync   │
└──────────────────────────┬──────────────────────────┘
                           │ AC secret name flows to service CR
                           ▼
┌─────────────────────────────────────────────────────┐
│  service operators                                  │
│  - Watch spec.auth.applicationCredentialSecret      │
│  - Add *-ac-consumer finalizer on consumed Secret   │
│  - Track status.applicationCredentialSecret for     │
│    rotation handoff                                 │
│  - Render config with v3applicationcredential       │
└─────────────────────────────────────────────────────┘
```

Shared helpers live in `keystone-operator/api/v1beta1` (`ManageACSecretFinalizer`, `RemoveACSecretConsumerFinalizer`, `GetACCRName`, `GetServiceNameFromACCR`, `IsEDPMService()`, secret key constants `ACIDSecretKey`/`ACSecretSecretKey`, and `EDPMServiceAnnotation`).

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

- **Placement** lives in `nova-operator` but has its own independent AC CR (`ac-placement`), per-service override (`spec.placement.applicationCredential`), and consumer finalizer (`openstack.org/placementapi-ac-consumer`).
- **Telemetry** has three independent sub-services (Aodh, Ceilometer, CloudKitty), each with its own AC CR and config section (`applicationCredentialAodh`, `applicationCredentialCeilometer`, `applicationCredentialCloudKitty`). Note that Aodh's AC CR is named `ac-aodh`, but the consuming Kubernetes CR is `Autoscaling`, so the consumer finalizer is `openstack.org/autoscaling-ac-consumer` (not `aodh-ac-consumer`).
- **Ironic** creates two AC CRs: one for `ironic` and one for `ironic-inspector` (separate consumer finalizers: `openstack.org/ironic-ac-consumer` and `openstack.org/ironic-inspector-ac-consumer`). The parent `Ironic` CR tracks two separate status fields (`status.applicationCredentialSecret` and `status.inspectorApplicationCredentialSecret`).
- **Glance** creates a single AC CR shared across GlanceAPI instances; each GlanceAPI uses a per-API consumer finalizer name (`openstack.org/glance-<apiName>-<apiType>-ac-consumer`).
- **EDPM consumers** are **Nova** and **Ceilometer**. Their AC CRs are annotated `keystone.openstack.org/edpm-service: "true"`. Credentials are rendered into config consumed on dataplane nodes; see [EDPM awareness](#edpm-awareness).

## Workflow

### AC Creation

1. `openstack-operator` reconciles the `OpenStackControlPlane` CR.
2. For each service with AC enabled, it calls `EnsureApplicationCredentialForService`.
3. If the service is ready and no AC CR exists, it creates a `KeystoneApplicationCredential` CR (for example, `ac-heat`) owned by the control plane. It sets the EDPM annotation (`true` for Nova/Ceilometer, `false` for control-plane-only services).
4. `keystone-operator` reconciles the AC CR:
   - Authenticates as the service user using the password from `osp-secret`.
   - Creates an Application Credential in Keystone.
   - Creates an **immutable** Kubernetes Secret named `ac-<service>-<first5ofACID>-secret` containing `AC_ID` and `AC_SECRET`.
   - Labels the Secret (`application-credentials`, `application-credential-service`).
   - Adds `openstack.org/ac-secret-protection` and an owner reference to the AC CR.
   - Sets `status.secretName`, `status.acID`, `status.expiresAt`, and related timestamps.
5. The `openstack-operator` `Owns` the AC CR, so when `keystone-operator` updates its status (sets Ready, populates `status.secretName`), controller-runtime triggers a reconcile of the owning `OpenStackControlPlane`. The `openstack-operator` reads `status.secretName` and sets `spec.auth.applicationCredentialSecret` on the service template (for example, Heat).
6. The service operator watches that Secret, adds its consumer finalizer, renders `v3applicationcredential` config, and records `status.applicationCredentialSecret` when consumption is established.

### AC Rotation

Rotation is triggered by the `keystone-operator` AC controller during reconcile.

#### Rotation triggers

The controller first evaluates `needsRotation()`, which returns true in these cases:

1. **No AC exists yet** — `status.acID` is empty (initial creation).
2. **Security-critical fields changed** — `securityHash` (from `roles`, `accessRules`, `unrestricted`) differs from the stored hash (immediate rotation).
3. **Grace period reached** — `time.Now()` is after `expiresAt - gracePeriodDays`.

Additionally, if `status.secretName` is set but that Secret no longer exists, the controller forces rotation/recreation.

#### Rotation behavior (keystone-operator)

On rotation:

- Creates a **new** AC in Keystone (name includes a random suffix).
- Creates a **new** immutable Secret with a unique name (for example, `ac-heat-d38dc-secret`).
- Moves the previous `status.secretName` to `status.previousSecretName` and updates `status.secretName` to the new Secret.
- Emits `ApplicationCredentialRotated` on the AC CR (not on initial create).
- Runs `cleanupUnusedRotatedSecrets` for older Secrets (see [Lifecycle and cleanup](#lifecycle-and-cleanup)).

#### Propagation (openstack-operator → service operators)

1. `openstack-operator` sees the new `status.secretName` and updates the service CR **spec** (`applicationCredentialSecret`).
2. The service operator detects the spec change and reconciles toward the new Secret.

#### Consumer finalizer handshake (service operators)

Service operators that implement AC consumer finalizers use a two-phase pattern (parent CR or API CR, depending on the service):

1. **Early** — add the service consumer finalizer (for example, `openstack.org/heat-ac-consumer`) to the **new** Secret from spec before or during rollout, so keystone-operator does not revoke it prematurely.
2. **Late** — after all relevant sub-services report ready with the new credentials, remove the consumer finalizer from the **old** Secret named in `status.applicationCredentialSecret`, then set `status.applicationCredentialSecret` to match spec.

During rotation, **spec** holds the desired (new) Secret; **status** holds the old Secret that still has the consumer finalizer until handoff completes.

Why the split matters: `status.applicationCredentialSecret` is the controller's memory of which secret it is still protecting. Setting `status = spec` immediately would erase the old secret name, and the controller would never know which secret to remove the finalizer from. The status update is always persisted (via `helper.PatchInstance` in `defer`), so the concern is not persistence but **timing** — status must lag behind spec until the old secret is safe to release.

For the non-rotation case (initial adoption, or spec and status already match), `status` is set to match `spec` immediately since there is no old secret to protect.

On service CR deletion, the operator removes consumer finalizers from both `status.applicationCredentialSecret` and `spec.auth.applicationCredentialSecret` (covers a crash between adding a finalizer and updating status).

#### End-to-end rotation flow

```text
1. keystone-operator: rotation needed → new AC + new immutable Secret
2. keystone-operator: status.secretName updated; previousSecretName set
3. openstack-operator: service spec.applicationCredentialSecret → new Secret
4. service operator: finalizer on new Secret; deploy with new credentials
5. service operator: when ready, remove finalizer from old Secret; status ← spec
6. keystone-operator: cleanupUnusedRotatedSecrets revokes/deletes orphans
   (skips current, previous, and any Secret with a *-ac-consumer finalizer)
```

### Manual rotation and test triggers

#### Immediate rotation (spec change)

Change security-critical fields (`roles`, `accessRules`, or `unrestricted`) on the `OpenStackControlPlane` CR (global `applicationCredential` or per-service override). The `openstack-operator` propagates these to the AC CR, the security hash changes, and keystone-operator triggers rotation on the next reconcile. The AC CR itself is owned by `openstack-operator` and should not be patched directly — manual spec changes are reconciled back.

#### Trigger via `status.expiresAt`

Patch `status.expiresAt` into the past so the credential falls inside the grace window:

```bash
oc patch -n openstack keystoneapplicationcredential ac-heat \
  --type=merge --subresource=status \
  -p '{"status":{"expiresAt":"2001-05-19T00:00:00Z"}}'
```

This uses the normal rotation path (new immutable Secret with a unique name). Unlike deleting the Secret, this approach keeps the existing Secret present, so service operators do not transiently reconcile into `ErrACSecretNotFound`.

#### Recovery: missing Secret

If the referenced Secret is deleted (only after bypassing finalizer `openstack.org/ac-secret-protection`), keystone-operator detects the missing Secret and recreates credentials. Service operators may transiently hit `ErrACSecretNotFound` until the new Secret exists.

#### Deleting the AppCred CR

The AC CR is owned by `openstack-operator`. Deleting it while the service remains enabled causes `openstack-operator` to recreate it on the next reconcile. During the recreation window, the service CR spec retains the old secret name (openstack-operator returns without updating the template while the new AC CR is not yet ready), and the old Kubernetes Secret remains present because its consumer finalizer (for example, `openstack.org/heat-ac-consumer`) is still there -- the service operator only removes that finalizer after the new secret is propagated to spec and all sub-services are ready with the new credentials. However, the consumer finalizer only protects the **Kubernetes Secret object**; it does not prevent keystone-operator from revoking the **Keystone-side application credential** during `reconcileDelete` (when AC CR is being deleted). After revocation, the AC_ID/AC_SECRET values in the Secret are no longer valid in Keystone, so services may see authentication errors once their cached Keystone token expires during this window. Once the new AC CR is ready, the new secret propagates to the service CR and triggers a single pod restart.

AC CR deletion happens automatically when AC is disabled (globally or per-service) or the service itself is disabled. When `keystone-operator` processes the delete: for EDPM-aware CRs, deletion is deferred until NodeSet secret hashes are in sync; then it **best-effort revokes** Keystone ACs for labeled Secrets, removes protection finalizers, and allows garbage collection.

### Lifecycle and cleanup

#### Unused rotated secrets

`cleanupUnusedRotatedSecrets` lists Secrets for the service (`application-credentials` + `application-credential-service` labels). For each Secret that is **not** `status.secretName`, **not** `status.previousSecretName`, and has **no** `openstack.org/*-ac-consumer` finalizer:

- Revokes the AC in Keystone (best-effort)
- Removes `openstack.org/ac-secret-protection`
- Deletes the Kubernetes Secret

Secrets still referenced by a service consumer finalizer are never deleted by this path.

#### Expiration vs Kubernetes finalizers

Under normal operation, rotation occurs during the grace window well before the current AC expires, and the consumer finalizer handshake ensures old credentials are not cleaned up until all services have switched to the new ones. In the unlikely event that rotation or handoff stalls (for example, keystone-operator is down for an extended period), the AC will still expire in Keystone on its configured schedule. A consumer finalizer only prevents **Kubernetes Secret** cleanup; an expired AC cannot authenticate even if the Secret still exists.

#### AC CR deletion

When the AC CR is deleted (AC disabled or service disabled):

- EDPM-aware CRs wait for `OpenStackDataPlaneNodeSet` secret hash sync (same as cleanup).
- Keystone revocation is attempted for all AC Secrets for that service (by label).
- Protection finalizers are removed so owner-reference GC can delete Secrets.

### How service operators consume ACs

- **Field indexer** on `.spec.auth.applicationCredentialSecret` (or service-specific path) for Secret → CR mapping.
- **Secret watch** (often `ResourceVersionChangedPredicate`) to reconcile on Secret changes.
- **Config generation** reads `AC_ID` / `AC_SECRET` via `keystonev1.ACIDSecretKey` and `keystonev1.ACSecretSecretKey`.
- **AC not configured** — empty `applicationCredentialSecret` on the service CR → password auth in templates.
- **AC Secret missing** — spec set but Secret absent → reconcile error (`ErrACSecretNotFound`); running pods keep prior config until rolled.

#### AC lifecycle ownership patterns

Depending on the operator, AC consumer finalizer management and `status.applicationCredentialSecret` tracking live on different CRs. This affects which CR to inspect during troubleshooting:

| Pattern | Operators |
|---------|-----------|
| **Parent CR** (top-level CR manages AC lifecycle) | Heat, Barbican, Nova, Watcher, Ironic, Cinder, Manila |
| **Sub-CR / API CR** (AC lifecycle on the API sub-CR) | GlanceAPI, NeutronAPI, OctaviaAPI, DesignateAPI, SwiftProxy, PlacementAPI |
| **Independent CRs** (each is its own top-level CR) | Autoscaling (Aodh), Ceilometer, CloudKitty |

## EDPM awareness

Nova and Ceilometer render AC data into secrets deployed to EDPM dataplane nodes. After control-plane rotation, nodes keep old credentials until the next dataplane deployment updates those secrets.

**Annotation:** `openstack-operator` sets `keystone.openstack.org/edpm-service` on each AC CR (`"true"` for Nova and Ceilometer, `"false"` for control-plane-only services). If the annotation is missing, keystone-operator defaults to EDPM-aware behavior (fail-safe).

**NodeSet hash sync:** Before `cleanupUnusedRotatedSecrets` or AC CR deletion, keystone-operator calls `edpm.AreSecretHashesInSync()` (`lib-common/modules/edpm/unstructured`), comparing each `OpenStackDataPlaneNodeSet` `status.secretHashes` to live secret hashes. If any NodeSet is stale, cleanup and delete are deferred.

**Watch:** The AC controller watches `OpenStackDataPlaneNodeSet` status changes (for example, after EDPM deploy updates hashes) and re-evaluates.

**Operator action:** Monitor `ApplicationCredentialRotated` events and redeploy affected NodeSets during the grace period so dataplane config picks up new credentials before the old AC expires in Keystone.

Control-plane-only services (Heat, Barbican, Cinder on CP, etc.) skip the NodeSet check when `edpm-service: "false"`.

## Control plane operations and observability

### List Application Credentials

```bash
oc get appcred -n openstack
```

Example output:

```text
NAME                  ACID                               SECRETNAME                          LASTROTATED            ROTATIONELIGIBLE       STATUS   MESSAGE
ac-barbican           d38dc4310fbf4601bbe9f4234eb24114   ac-barbican-d38dc-secret            2026-03-12T08:23:58Z   2026-03-15T08:23:58Z   True     Setup complete
ac-ceilometer         0e0e7c8f243c4ce8913fd709d44c0104   ac-ceilometer-0e0e7-secret                                 2028-02-29T13:35:06Z   True     Setup complete
ac-cinder             b8c7fb9d3abc4ce18727a56f870c9a18   ac-cinder-b8c7f-secret                                     2027-03-03T13:04:41Z   True     Setup complete
ac-glance             da20e5b59d0f4227938046c60857cb62   ac-glance-da20e-secret              2026-03-12T08:31:42Z   2027-03-13T08:31:42Z   True     Setup complete
ac-ironic             ec6bbfb36b8041cbad96f4360a84cd71   ac-ironic-ec6bb-secret                                     2027-03-03T13:04:42Z   True     Setup complete
ac-ironic-inspector   81aebe21c2b94a2b89caec4b1fe652c1   ac-ironic-inspector-81aeb-secret                           2027-03-03T13:04:45Z   True     Setup complete
ac-octavia            0dd0b1b7247743b980d909334a4ea8ad   ac-octavia-0dd0b-secret                                    2027-03-07T09:39:19Z   True     Setup complete
```

Secret names follow `ac-<service>-<5-char-id-prefix>-secret`, not a fixed `ac-<service>-secret` suffix.

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
  annotations:
    keystone.openstack.org/edpm-service: "false"
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
  secretName: ac-barbican-d38dc-secret
  previousSecretName: ac-barbican-7b23d-secret
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

On rotation, `keystone-operator` emits a Kubernetes event on the AppCred CR with reason `ApplicationCredentialRotated`:

```bash
oc get events -n openstack --sort-by=.lastTimestamp | grep ApplicationCredentialRotated
```

Example output:

```text
8s  Normal  ApplicationCredentialRotated  keystoneapplicationcredential/ac-barbican  Rotated credentials for user barbican. New expiration: 2026-03-18T08:24:18Z, next rotation eligible: 2026-03-16T08:24:18Z (grace period: 2 days). Previous credential expires: 2026-03-17T08:23:58Z
```

For EDPM consumers (Nova and Ceilometer), these events are particularly important as a signal to plan `OpenStackDataPlaneNodeSet` redeployment before the previous credential's Keystone expiration.

### List AC secrets for a service

```bash
oc get secret -n openstack -l application-credential-service=heat
```

Check consumer finalizers:

```bash
oc get secret -n openstack ac-heat-d38dc-secret -o jsonpath='{.metadata.finalizers}'
```

## Conditions and states

The AppCred CR does not expose a dedicated phase/status enum such as `Creating`, `Rotating`, or `Failed`. Instead, it exposes conditions and timestamps.

The main conditions are:

- `Ready`
- `KeystoneAPIReady`
- `KeystoneApplicationCredentialReady`

Typical interpretations:

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

## Limitations and operational notes

1. **EDPM manual redeploy** — Dataplane nodes do not pick up rotated ACs automatically. Nova and Ceilometer require `OpenStackDataPlaneNodeSet` deployment during the grace window. Keystone defers revocation/cleanup while NodeSet hashes are out of sync, but Keystone expiration still applies to credentials in use on stale nodes.

2. **`status.applicationCredentialSecret` on service CRs** — Used for rotation handoff (which old Secret still has a consumer finalizer). It is persisted in etcd during normal operation but is not specially handled for control-plane disaster recovery. Restore without status can skip old-secret cleanup and leave orphaned consumer finalizers (Secrets remain until manually fixed). Delete reconciliation still checks both status and spec.

3. **Orphaned consumer finalizers** — A stale `*-ac-consumer` finalizer blocks keystone-operator from deleting that Secret. It does not keep the Keystone AC valid past `expires_at`. Operators may remove the finalizer only after confirming nothing still uses that Secret.

4. **Manual Keystone cleanup** — For suspected compromise, operators can revoke in Keystone directly:

   ```bash
   openstack application credential delete <AC_ID>
   ```

   Do not revoke credentials still in use on EDPM or control plane without coordinating rollout.

## Related documentation

- [keystone-operator/docs/applicationcredentials.md](https://github.com/openstack-k8s-operators/keystone-operator/blob/main/docs/applicationcredentials.md) — AC controller-focused reference (API fields, controller steps, EDPM tests).
