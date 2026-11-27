# PQC: inheriting the cluster TLS security profile

OpenShift lets an administrator set a cluster-wide TLS security profile on the
`APIServer` CR. Core components honour it; OpenStack services historically did
not, because they serve TLS from a static `ssl.conf` shipped by `lib-common`
with hardcoded cipher and protocol values.

This document describes how RHOSO closes that gap: the **openstack-operator
publishes** the resolved profile, and **service operators consume** it. The
mechanism is the first step of the PQC work: it gives every service the ability
to follow the cluster policy, which is what later enables TLS 1.3-only endpoints
and hybrid post-quantum key exchange.

## Overview

```
  APIServer/cluster  (config.openshift.io/v1)
  spec.tlsSecurityProfile: Old | Intermediate | Modern | Custom
        │
        │  read (only the openstack-operator has RBAC for it)
        ▼
  openstack-operator  ──  spec.tls.profile: None | Inherit | Intermediate | Modern
  ReconcileTLSProfile
        │
        │  resolves the profile into TLS configuration
        ▼
  ConfigMap  openstack-ssl-profile   (in the OpenStackControlPlane namespace)
        SSLProtocol      SSLCipherSuite      MinTLSVersion
        │
        │  lib-common: ApplyTemplateDefaults
        ▼
  service operator  ──  renders ssl.conf into the service ConfigMap/Secret
        │
        ▼
  httpd (and other TLS consumers)
```

The interface between the two sides is a single ConfigMap with a well-known
name. Neither side needs to know about the other: the producer writes it, the
consumer reads it if it is there.

## The interface

| | |
|---|---|
| Object | `ConfigMap` named `openstack-ssl-profile`, in the control plane namespace |
| Constant | `util.TLSProfileConfigMap` (`lib-common/modules/common/util`) |
| Lifecycle | owned by `openstack-operator`, garbage collected with the control plane |

Keys:

| Key | Meaning | Consumers |
|---|---|---|
| `SSLProtocol` | mod_ssl directive, e.g. `all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1` | httpd |
| `SSLCipherSuite` | OpenSSL cipher list | httpd |
| `MinTLSVersion` | minimum version in OpenShift spelling, e.g. `VersionTLS12` | non-httpd backends |

`MinTLSVersion` exists because the other two are httpd syntax. Galera, RabbitMQ,
OVN cannot use `SSLProtocol`; each needs the same fact in its own dialect, and
the openstack-operator is the only component with RBAC on `apiservers`, so it is
the only one that can learn it.

It is a `ConfigMap` rather than a Secret because the content derives from a
cluster-readable API object and is not sensitive, and it should be inspectable
with `oc get cm`.

## Producer side: openstack-operator

The producer is opt-in through `spec.tls.profile` on `OpenStackControlPlane`:

| Value | Behaviour |
|---|---|
| `None` (default) | nothing is published; services keep their built-in defaults. The ConfigMap is removed and the condition is not registered. |
| `Inherit` | read `APIServer/cluster` and publish whatever it declares. A `Custom` profile set there is honoured; its TLS 1.3 suite selection reaches httpd when the profile sets `minTLSVersion: VersionTLS13`, and below that the TLS 1.3 suites stay at OpenSSL's defaults (see below). |
| `Intermediate` | publish the OpenShift Intermediate profile, regardless of the cluster |
| `Modern` | publish the OpenShift Modern profile, regardless of the cluster |

**Note**
>Keeping `None` as the default means a minor update does not change TLS
>behaviour: the feature is explicitly enabled by an administrator.

Resolution uses the profile tables shipped by `openshift/api`
(`configv1.TLSProfiles`). The producer then converts the minimum TLS version to
the `SSLProtocol` syntax.

### Why the TLS 1.3 suites need the `TLSv1.3` keyword

OpenShift puts both cipher families in one `ciphers` list: `TLS_*` names for
TLS 1.3, `ECDHE-*` / `DHE-*` names for TLS 1.2 and below. OpenSSL keeps them in
two separate lists, and a bare `SSLCipherSuite` only reaches the first one. It
skips the `TLS_*` entries, and fails the whole directive once nothing else is
left — which is exactly what a TLS 1.3-only profile such as `Modern` produces:

```
AH01898: Unable to configure permitted SSL ciphers
SSL Library Error: error:0A0000B9:SSL routines::no cipher match
AH02311: Fatal error initialising mod_ssl, exiting.
```

httpd 2.4.43 added an optional protocol parameter to the directive, so the two
families are configured independently. The negotiated version decides which
list is used; neither line overrides the other:

```
SSLCipherSuite <tls1.2 names>           -> SSL_CTX_set_cipher_list
SSLCipherSuite TLSv1.3 <TLS_* names>    -> SSL_CTX_set_ciphersuites
```

The producer therefore keeps every name the profile declares, and prefixes the
value with the `TLSv1.3` keyword whenever the profile mandates `VersionTLS13`.
Below that the list goes out unprefixed. It still carries the `TLS_*` names,
which OpenSSL ignores in that position, so the profile keeps OpenSSL's default
TLS 1.3 suites — but only as long as the list also holds at least one TLS 1.2
name for the directive to use. For the profiles OpenShift ships that always
holds, and nothing is lost, since their TLS 1.3 sets are exactly those
defaults.

What each profile resolves to:

| Profile | `MinTLSVersion` | `SSLProtocol` | `SSLCipherSuite` |
|---|---|---|---|
| `Old` | `VersionTLS10` | `all -SSLv2 -SSLv3` | declared list, unprefixed |
| `Intermediate`, and the fallback when the cluster declares no profile | `VersionTLS12` | `all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1` | declared list, unprefixed |
| `Modern` | `VersionTLS13` | `all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1 -TLSv1.2` | `TLSv1.3 ` + the three `TLS_*` suites |
| `Custom` below `VersionTLS13`, list holds at least one TLS 1.2 name | as declared | derived from it | declared list, unprefixed |
| `Custom` below `VersionTLS13`, list holds only `TLS_*` names | not published | not published | **profile rejected**, see below |
| `Custom` at `VersionTLS13`, list holds at least one `TLS_*` suite | `VersionTLS13` | `all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1 -TLSv1.2` | `TLSv1.3 ` + the declared list; TLS 1.2 names in it are ignored |
| `Custom` at `VersionTLS13`, list holds only TLS 1.2 names | not published | not published | **profile rejected**, see below |

The cipher key is left out altogether when the resolved list is empty, so
`ssl.conf` stays on its own fallback instead of rendering an empty directive.

Rendered `ssl.conf`:

```
# Intermediate
SSLCipherSuite TLS_AES_128_GCM_SHA256:...:DHE-RSA-AES256-GCM-SHA384
SSLProtocol    all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

# Modern
SSLCipherSuite TLSv1.3 TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
SSLProtocol    all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1 -TLSv1.2
```

### A `Custom` list that does not match its own minimum

Whichever form of the directive the minimum selects, OpenSSL *skips* the names
that do not belong in that position rather than rejecting them, and fails the
directive only once no usable name is left. A list that mixes the two families
is therefore fine: the names that cannot apply are dropped, and at
`VersionTLS13` they could not have applied anyway, since `SSLProtocol` already
excludes everything below TLS 1.3.

A list holding *nothing* valid for its own minimum is a different matter. It
would reach httpd as a directive OpenSSL cannot fill, and every service that
rendered the file would fail to start with the `AH02311` error above:

- `minTLSVersion` below `VersionTLS13` with **only** `TLS_*` suites. The
  unprefixed directive goes to `SSL_CTX_set_cipher_list`, which skips every
  name in it. This is the failure that was seen on a cluster.
- `minTLSVersion: VersionTLS13` with **no** `TLS_*` suite at all. The prefixed
  directive goes to `SSL_CTX_set_ciphersuites`, which skips every TLS 1.2 name
  and is left with nothing.

The producer rejects both rather than publishing them. The profile does not
resolve, `OpenStackControlPlaneTLSProfileReady` goes `False` with
`SeverityError`, and no ConfigMap is written, so a profile that would take
every endpoint down instead leaves the services on the last profile that did
resolve, with the reason in the control plane status rather than in the logs
of every httpd container.

Only a hand-written `Custom` profile can reach either case. The lists and
minimums of the profiles OpenShift ships are fixed in
[`types_tlssecurityprofile.go`](https://github.com/openshift/api/blob/master/config/v1/types_tlssecurityprofile.go),
where `Old` and `Intermediate` always carry TLS 1.2 names alongside the `TLS_*`
ones and `Modern` is `TLS_*` only at `VersionTLS13`. Nothing in that API ties
`ciphers` to `minTLSVersion` — the field takes any list of strings — and
upstream warns on it that *"invalid configurations can be catastrophic"*.

The OpenSSL behaviour behind all this can be checked against the build a
service image ships, without a cluster. The output below is from OpenSSL
4.1.0; the first result was also seen on 3.5.8, and the second matches the
`no cipher match` failure observed on a cluster running 3.5.9:

```
$ openssl ciphers -s -tls1_3 \
    -ciphersuites 'TLS_AES_256_GCM_SHA384:ECDHE-RSA-AES256-GCM-SHA384'
TLS_AES_256_GCM_SHA384                       # mixed: the TLS 1.2 name is dropped

$ openssl ciphers -s -tls1_3 -ciphersuites 'ECDHE-RSA-AES256-GCM-SHA384'
Error setting TLSv1.3 ciphersuites           # nothing valid left
...:error:0A0000B9:SSL routines:set_ciphersuites:no cipher match:ssl/ssl_ciph.c
```

### Status

Status is reported through the `OpenStackControlPlaneTLSProfileReady`
condition, which only exists while the feature is enabled:

- Not set at all when `spec.tls.profile` is `None` (the default): neither the
  ConfigMap nor the condition exists, and `status.tls.profile` is absent.
- `True` once the ConfigMap is published.
- `True` with a message explaining that built-in defaults apply when `Inherit`
  is selected but `APIServer/cluster` does not exist. Any previously published
  ConfigMap is withdrawn and `status.tls.profile` is cleared.
- `False` with `SeverityError` when the profile cannot be resolved: for instance
  `type: Custom` with no `custom:` block. Any ConfigMap published earlier is
  deliberately left in place, so a mistyped profile cannot silently widen every
  service back to the built-in defaults; the condition message says as much.

`status.tls.profile` describes the last profile successfully published in the
ConfigMap. For example, when `Inherit` resolves to the cluster's `Intermediate`
profile:

```yaml
status:
  tls:
    profile:
      type: Intermediate
      source: Cluster
      minTLSVersion: VersionTLS12
```

The fields describe the producer's output:

| Field | Meaning |
|---|---|
| `type` | Resolved profile: `Old`, `Intermediate`, `Modern`, or `Custom`; never `Inherit` or `None` |
| `source` | `Cluster` when inherited from `APIServer/cluster`, or `Spec` when selected directly |
| `minTLSVersion` | Minimum TLS version published in the ConfigMap |

Selecting `Modern` directly reports type `Modern`, source `Spec`, and
`minTLSVersion: VersionTLS13`. Inheriting a hand-written `Custom` profile
reports type `Custom`, source `Cluster`, and that profile's minimum version.

The status block updates only after a successful ConfigMap write. A resolution
or publication error leaves the last published status in place, matching the
ConfigMap retained by the operator. Before the first successful publication,
the block is absent. Setting `spec.tls.profile` back to `None` withdraws the
ConfigMap and removes both the status block and the condition.

This reports what the operator **published**. Services consume the ConfigMap
asynchronously, so their pods may still be rolling to the new configuration.
It does not confirm what each pod enforces.

Example spec enabling inheritance:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: myctlplane
spec:
  tls:
    profile: Inherit   # None (default) | Inherit | Intermediate | Modern
```

## Consumer side

### lib-common: merging the defaults

`lib-common` merges the ConfigMap underneath the `ConfigOptions` of every
`Template` it renders. Both `EnsureConfigMaps` and `EnsureSecrets` call
`util.ApplyTemplateDefaults(ctx, h, templates, util.DefaultTemplateConfigMaps)`
before rendering; a missing ConfigMap contributes nothing, so clusters without a
published profile behave exactly as before. `TemplateTypeCustom` templates are
skipped, since they are not rendered.

Precedence, highest first:

| Priority | Source | Controlled by |
|---|---|---|
| 1 | `extraMounts` / user-supplied config | end user |
| 2 | the operator's own `Template.ConfigOptions` | service operator |
| 3 | `openstack-ssl-profile` | openstack-operator (cluster profile) |
| 4 | hardcoded defaults in the template | lib-common |

**Published does not imply enforced.** Higher-priority `Template.ConfigOptions`
or user-supplied configuration through `extraMounts` can override the published
settings. These overrides are not reflected in `status.tls.profile` or the
`OpenStackControlPlaneTLSProfileReady` condition: both describe publication by
the producer. Users supplying overrides are responsible for their contents and
for verifying the effective TLS configuration at the service endpoint.

`ssl.conf` is a Go template with its own fallbacks, so a service that publishes
nothing still gets a sane configuration:

```
SSLCipherSuite ECDHE+AESGCM:DHE+AESGCM:!aNULL:!MD5:!RC4:!3DES
SSLProtocol    all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
```

### What a service operator has to do

Consuming the profile is explicit. A service opts in by doing four things.

1. **Ask for the template.** Set `CommonTemplates: []string{"ssl.conf"}` on the
   `Template` the service renders, so lib-common includes the shared file.
2. **Include the rendered file.** Mount it where httpd reads it (typically
   `/etc/httpd/conf.d/ssl.conf`) and make sure the service's httpd configuration
   includes that directory.
3. **Watch the ConfigMap.** Register the generic helper in `SetupWithManager`:

   ```go
   Watches(&corev1.ConfigMap{},
       util.WatchDefaultTemplateConfigMap(r.Client, &glancev1.GlanceAPI{})),
   ```

   Without it, a profile change would only be picked up on the next unrelated
   reconcile of the service.
4. **Grant read access.** The RBAC marker on the controller that registers the
   watch must include `get`, `list` and `watch` on `configmaps`. `get` is for the
   template-defaults read, `list`/`watch` for the informer behind the watch.

Once the rendered `ssl.conf` changes, the hash of the service's config ConfigMap
or Secret changes and the deployment/statefulset rolls, so new pods start httpd
with the new values.

### Reconciliation and propagation

`WatchDefaultTemplateConfigMap` enqueues every CR of the calling controller's
kind in the ConfigMap's namespace, so a change to `openstack-ssl-profile`
re-renders and rolls every affected service.

A change propagates in two hops: openstack-operator republishes the ConfigMap,
then each service operator reconciles and rolls its pods.

## Verifying on a cluster

Checked against `httpd-2.4.63-18.el10` with `openssl-libs-3.5.9-1.el10`. Any
service that mounts the rendered `ssl.conf` works; `glance` is used here.

**1. What the operator published.**

```
$ oc get cm openstack-ssl-profile -o jsonpath='{.data}'
$ oc get openstackcontrolplane myctlplane -o jsonpath='{.status.tls.profile}'
```

**2. The suite the profile allows is the one negotiated.** From a pod that can
reach the endpoint:

```
$ openssl s_client -connect glance-default-internal.openstack.svc:9292 \
    -tls1_3 -ciphersuites TLS_AES_128_GCM_SHA256
CONNECTED(00000003)
depth=1 CN=rootca-internal
verify return:1
depth=0
verify return:1
---
Certificate chain
 0 s:
   i:CN=rootca-internal
   a:PKEY: RSA, 2048 (bit); sigalg: sha384WithRSAEncryption
---
Server certificate
-----BEGIN CERTIFICATE-----
<redacted>
-----END CERTIFICATE-----
subject=
issuer=CN=rootca-internal
---
Peer signing digest: SHA256
Peer signature type: rsa_pss_rsae_sha256
Negotiated TLS1.3 group: X25519MLKEM768
---
SSL handshake has read 2747 bytes and written 1568 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Protocol: TLSv1.3
Verify return code: 0 (ok)
```

`Protocol` and `Cipher` are the two lines that matter here: they name the
version and the suite this handshake settled on, which is what the profile
controls.

`Negotiated TLS1.3 group: X25519MLKEM768` reports the key exchange group,
which this mechanism does not configure at all. A hybrid post-quantum group is
used only when the client offers it and the server accepts it: a client
offering only classical groups completes the same TLS 1.3 handshake with a
classical group, and the line would name that one instead. So the line
confirms the group for this one connection; it does not say the endpoint gives
every client post-quantum key exchange.

**3. The restriction actually narrows.** This is the check that distinguishes
an enforced TLS 1.3 list from OpenSSL's defaults. Against a profile that allows
only `TLS_AES_256_GCM_SHA384`, asking for a different suite has to fail the
handshake:

```
$ openssl s_client -connect glance-default-internal.openstack.svc:9292 \
    -tls1_3 -ciphersuites TLS_AES_128_GCM_SHA256   # must fail
$ openssl s_client -connect glance-default-internal.openstack.svc:9292 \
    -tls1_3 -ciphersuites TLS_AES_256_GCM_SHA384   # must succeed
```

If the first one succeeds, the suites were never restricted: the list reached
`SSL_CTX_set_cipher_list` and was ignored, rather than reaching
`SSL_CTX_set_ciphersuites`.

## Note

**Non-httpd backends.** The neutral `MinTLSVersion` key is published so that
these services can follow it. Each consumer can translate this information
into its own configuration parameters.

## References

- OpenShift TLS security profile types, the `Old` / `Intermediate` / `Modern`
  cipher tables, and the warning on `Custom`:
  <https://github.com/openshift/api/blob/master/config/v1/types_tlssecurityprofile.go>
- Apache `SSLCipherSuite`, including the optional protocol parameter that
  selects the TLS 1.3 list:
  <https://httpd.apache.org/docs/2.4/mod/mod_ssl.html#sslciphersuite>
- The httpd changelog entry introducing that parameter in 2.4.43, and stating
  that the two cipher sets are configured separately:
  <https://svn.apache.org/repos/asf/httpd/sandbox/httpd-2.4.x-trunk-delta/CHANGES>
