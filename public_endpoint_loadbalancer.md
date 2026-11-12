# Exposing public endpoints with a LoadBalancer, custom hostnames, ports and certificates

By default the public OpenStack API endpoints are exposed through OpenShift `Route`s (host `<svc>-<namespace>.apps.<cluster-domain>`, port 443, certificate from the router or the `rootca-public` CA). This document describes how to expose them through a MetalLB `LoadBalancer` Service instead, and how to control the **hostname**, the **port** and the **TLS certificate** clients see.

Typical reasons:

* clients, firewalls and DNS entries are pinned to a fixed VIP, hostname and port, e.g. when moving from RHOSP 17.1 and keeping the old external VIP,
* the public endpoint should live on a dedicated external network instead of behind the OpenShift ingress,
* a certificate from the company CA must be presented for the endpoint.

Everything is configured in the `OpenStackControlPlane` CR. Routes stay the default; nothing in this document is required for a standard deployment.

For the background of MetalLB, `NetConfig` and the `IPAddressPool` see [Networking](networking.md), for the general certificate handling see [TLS](tls.md).

## Prerequisites

1. **The public network on the OpenShift worker nodes.** The worker nodes that host the control plane must be connected to the network the public endpoint should be reachable on (a VLAN or interface with an address in that subnet). If the network is not yet configured on the nodes, add it with a `NodeNetworkConfigurationPolicy` (NNCP, NMstate operator) before anything else, see [Networking](networking.md#connect-ocp-workers-to-the-isolated-networks). MetalLB in layer 2 mode announces the IP only from nodes that have an interface on that network.
2. **A MetalLB `IPAddressPool` and `L2Advertisement`** on that network (see [Networking](networking.md)). The examples use a pool called `public` and the IP `192.168.111.51`.
3. **A DNS name for every custom hostname.** The operator does **not** create DNS records for custom hostnames. Before (or right after) applying the configuration the hostname has to resolve to the LoadBalancer IP everywhere it is used, see [DNS](#dns-registering-the-hostnames).
4. **A certificate that matches the hostname** if you bring your own, see [Certificates](#certificates).
5. Service operators that are new enough to support this feature. A service whose operator is too old still gets the Service type, the certificate SANs and bring-your-own certificates, but may not serve the custom hostname or follow the `endpointURL` port. Check the release notes for the operator versions per service.

## Basic setup: public endpoint on a LoadBalancer

Per API service, switch the **public** Service to type `LoadBalancer` and add the MetalLB annotations. The openstack-operator then no longer creates a Route for that endpoint.

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack-control-plane
  namespace: openstack
spec:
  keystone:
    template:
      override:
        service:
          public:
            metadata:
              annotations:
                metallb.universe.tf/address-pool: public
                metallb.universe.tf/allow-shared-ip: public
                metallb.universe.tf/loadBalancerIPs: 192.168.111.51
            spec:
              type: LoadBalancer
```

Without anything else the endpoint is reachable on the LoadBalancer IP, on the service's native port (keystone `5000`), with the hostname `keystone-public.openstack.svc` in the catalog. That is rarely what you want, as clients cannot validate a certificate for it. Use `endpointURL` as described next.

The same pattern applies to the other services. Where the `override` lives depends on the service (e.g. `spec.nova.template.apiServiceTemplate.override`, `spec.glance.template.glanceAPIs.<name>.override`, `spec.cinder.template.cinderAPI.override`); the structure below `override` is identical. Horizon and the novncproxy are covered in [Services with a different layout](#services-with-a-different-layout).

## Custom hostname and port: `endpointURL`

`override.service.public.endpointURL` sets the URL clients use for the public endpoint. One setting drives four things:

| What | Result |
|---|---|
| Keystone catalog | the `public` endpoint is exactly this URL |
| httpd virtual host | the host becomes the public vhost `ServerName`; the in-cluster `<svc>-public.<ns>.svc[.<cluster-domain>]` names stay as aliases |
| Service certificate | the host is added as a SAN to the operator-issued certificate |
| LoadBalancer Service port | the front port becomes the port of the URL (see below) |

```yaml
spec:
  keystone:
    template:
      override:
        service:
          public:
            endpointURL: https://keystone.example.com:13000
            metadata:
              annotations:
                metallb.universe.tf/address-pool: public
                metallb.universe.tf/allow-shared-ip: public
                metallb.universe.tf/loadBalancerIPs: 192.168.111.51
            spec:
              type: LoadBalancer
```

Rules for `endpointURL`:

* It must contain a **fully qualified hostname**. A short name (`https://keystone:13000`) and IP addresses are rejected at admission. For a TLS-valid endpoint the client needs a name anyway.
* The scheme should be `https` (TLS is the default).
* A port in the URL is used as the **front port** of the LoadBalancer Service. The port the pod listens on (`targetPort`) is not changed. With no port in the URL the standard port of the scheme applies (`https` = 443).
* Without an `endpointURL` the Service keeps its native port.
* Do not additionally set `override.service.public.spec.ports[].port` to a different number than the URL port; this is rejected at admission. The URL is enough on its own.
* `endpointURL` only takes effect together with `type: LoadBalancer`. With the default `ClusterIP` Service the Route is still created and the catalog entry is set to the hostname of the Route, silently ignoring `endpointURL`.

Changing an existing Route-backed endpoint to a LoadBalancer removes the Route and the Route certificate the operator created, and keeps your `endpointURL`.

### Several public services on one IP

Several services can share one MetalLB IP (`allow-shared-ip` with the same value, same `loadBalancerIPs`) as long as **their ports differ**. Therefore put an **explicit port** in every `endpointURL` of services that share an IP: a URL without a port means 443, and two services on the same IP and port (for example with horizon on 443) conflict in MetalLB. Services on separate IPs are not affected.

When keeping the ports of an RHOSP 17.1 deployment (HAProxy terminated TLS on `13xxx` ports on the public VIP), set them in the URL:

| Service | RHOSP 17.1 public TLS port |
|---|---|
| keystone | 13000 |
| placement | 13778 |
| nova | 13774 |
| glance | 13292 |
| cinder | 13776 |
| neutron | 13696 |
| heat (api / cfn) | 13004 / 13005 |
| swift | 13808 |
| barbican | 13311 |
| octavia | 13876 |
| manila | 13786 |
| ironic (api / inspector) | 13385 / 13050 |
| designate | 13001 |
| aodh | 13042 |
| nova novncproxy | 13080 |
| horizon | 443 |

Services that did not exist in 17.1 (watcher, cyborg, cloudkitty, prometheus, alertmanager) have no such port. They keep their native port unless the URL says otherwise.

## DNS: registering the hostnames

The custom hostname is only a name; nothing resolves it until you register it. **This is a must-have**, otherwise clients, other OpenStack services and the data plane nodes cannot reach the endpoint by name and TLS validation fails. The operator does not manage these records: the public LoadBalancer Service has no `dnsmasq.network.openstack.org/hostname` annotation and no CNAME is created.

Register `keystone.example.com` → `192.168.111.51` (the `loadBalancerIPs` of the public Service; an `A` record, or a `CNAME` to a name that resolves to it) in every resolver that has to answer for it:

1. **Your upstream/corporate DNS server.** This is the authoritative place and what external clients (OpenStack CLI users, applications) use.
2. **The DNS server the OpenShift cluster uses upstream.** Pods in the cluster (including the OpenStack service pods that talk to a public endpoint, and the OpenStackClient pod) resolve through the cluster DNS, which forwards non-cluster names to it. Normally this is the same upstream server as in 1.; verify it is, e.g. from a pod:
   ```
   oc rsh openstackclient getent hosts keystone.example.com
   ```
3. **The dnsmasq instance of the control plane** (infra-operator `DNSMasq`) that the data plane nodes use. Do not add the custom hostnames to dnsmasq. Configure the upstream DNS server of 1. as the server dnsmasq forwards requests to, so it resolves the names through the upstream server:
   ```yaml
   apiVersion: network.openstack.org/v1beta1
   kind: DNSMasq
   metadata:
     name: dnsmasq
     namespace: openstack
   spec:
     options:
     - key: server
       values:
       - 192.168.122.1   # the upstream DNS server
   ```
   In the `OpenStackControlPlane` this is `spec.dns.template.options`. See [DNS](networking.md#dns).

Create one record per distinct custom hostname. If several services share one IP each service still needs its own name (or the same name with a different port, which is a valid, though uncommon, choice).

## Certificates

For a LoadBalancer public endpoint the certificate is presented by the API pod (httpd) itself. Which certificate is used is decided per endpoint:

| Endpoint | Certificate presented | How to bring your own |
|---|---|---|
| public, **LoadBalancer** | operator-issued (`rootca-public` or your custom issuer), with `endpointURL` host as SAN | `spec.<svc>.template.tls.api.public.secretName` |
| public, **Route** | router/Route certificate | `spec.<svc>.apiOverride.tls.secretName` (Route only) |
| **internal** | operator-issued (`rootca-internal`) | `spec.<svc>.template.tls.api.internal.secretName` |

In short: `tls.api.<endpoint>.secretName` is the certificate of the API pod; `apiOverride.tls.secretName` is only the certificate of the Route. With a Route `tls.api.public.secretName` is ignored (and logged).

### Default: operator-issued certificate

With `endpointURL` set nothing else is needed. cert-manager issues the public certificate from `rootca-public` with these SANs:

* the host of `endpointURL`,
* `<svc>-public.<ns>.svc` and `<svc>-public.<ns>.svc.<cluster-domain>`,
* anything listed in the `additionalSubjectNames` annotation of the Service.

Clients must trust `rootca-public`; it is part of the `combined-ca-bundle` secret of the control plane. Setting or changing `endpointURL` re-issues the certificate once.

To issue it from your own CA instead of `rootca-public`, use a custom issuer (`spec.tls.ingress.ca.customIssuer`), see [TLS](tls.md).

### Bring your own certificate

For a certificate you obtained yourself, create a secret in the control plane namespace with these keys:

* `tls.crt`: the certificate (add intermediate certificates after the leaf),
* `tls.key`: its private key,
* `ca.crt`: the CA that issued it.

```bash
oc create secret generic keystone-public-cert -n openstack \
  --from-file=tls.crt=keystone.example.com.crt \
  --from-file=tls.key=keystone.example.com.key \
  --from-file=ca.crt=company-ca.crt
```

and reference it for the endpoint:

```yaml
spec:
  tls:
    caBundleSecretName: my-company-ca       # secret with the CA(s), see below
  keystone:
    template:
      override:
        service:
          public:
            endpointURL: https://keystone.example.com:13000
            metadata:
              annotations:
                metallb.universe.tf/address-pool: public
                metallb.universe.tf/loadBalancerIPs: 192.168.111.51
            spec:
              type: LoadBalancer
      tls:
        api:
          public:
            secretName: keystone-public-cert
```

(`tls` here is the one in the service `template`, next to `override`; it is not the global `spec.tls`.)

Requirements and behavior:

* The certificate must be valid for the hostname of the `endpointURL` (SAN, not only CN). The operator only checks that `tls.crt` and `tls.key` are present, not their content.
* The CA that issued the certificate has to be in a secret referenced by `spec.tls.caBundleSecretName`, so that OpenStack services and EDPM nodes trust it. Secret format and behavior: [TLS](tls.md#combined-ca-bundle).
* While the secret does not exist the control plane retries every 10 seconds and reports `OpenStackControlPlaneCustomTLSReadyCondition=False`. An invalid secret is a reconcile error.
* When the **content** of the secret changes (renewal, rotation), the control plane reconciles and the service pods restart to load the new certificate. There is no need to touch the CR.
* If you remove the `secretName` again, the operator goes back to issuing the certificate itself.

### Extending the trusted CA bundle

Certificates from a CA that the control plane does not know (a company CA for a bring-your-own certificate or a custom issuer) must be trusted by the OpenStack services and the data plane nodes. Put the CA certificate(s) into a secret and reference it from `spec.tls.caBundleSecretName`; the operator merges them into `combined-ca-bundle`:

```bash
oc create secret generic my-company-ca -n openstack --from-file=ca.crt=company-ca.crt
```

```yaml
spec:
  tls:
    caBundleSecretName: my-company-ca
```

External clients (CLI users, applications) need the CA in their own trust store. See [TLS](tls.md#combined-ca-bundle).

## Services with a different layout

### Horizon

Horizon has a single Service override instead of a `public`/`internal` map, so `endpointURL` and the Service settings sit directly below `override.service`. The certificate secret is `tls.secretName` of the horizon template.

```yaml
spec:
  horizon:
    template:
      override:
        service:
          endpointURL: https://horizon.example.com
          metadata:
            annotations:
              metallb.universe.tf/address-pool: public
              metallb.universe.tf/loadBalancerIPs: 192.168.111.60
          spec:
            type: LoadBalancer
      tls:
        secretName: horizon-public-cert   # optional, bring your own certificate
```

`https://horizon.example.com` implies port 443 and is sufficient.

### nova novncproxy

The novncproxy is configured per cell, below `spec.nova.template.cellTemplates.<cell>.noVNCProxyServiceTemplate`. Like horizon it has a single Service override. It terminates TLS in the proxy itself (no httpd vhost); the certificate SAN and port handling are the same as described above. The certificate secret is `tls.service.secretName`.

```yaml
spec:
  nova:
    template:
      cellTemplates:
        cell1:
          noVNCProxyServiceTemplate:
            override:
              service:
                endpointURL: https://novncproxy.example.com:13080
                metadata:
                  annotations:
                    metallb.universe.tf/address-pool: public
                    metallb.universe.tf/allow-shared-ip: public
                    metallb.universe.tf/loadBalancerIPs: 192.168.111.61
                spec:
                  type: LoadBalancer
            tls:
              service:
                secretName: novncproxy-cell1-public-cert   # optional, bring your own certificate
```

Each cell with an enabled novncproxy needs its own hostname and `endpointURL`. With multiple cells, the public Services must not collide on the same IP and port: either use a **different port** per cell in the `endpointURL` (cells sharing one `loadBalancerIPs`), or a **different IP** per cell (then the same port is fine). A portless URL means 443 in every cell and therefore conflicts on a shared IP.

The separate vencrypt certificate (proxy to the compute node's QEMU) is not affected.

### Glance

With several public Glance APIs every API needs its own `endpointURL`; APIs sharing a VIP must use distinct ports.

## Supported services

Every service whose public endpoint is created by the openstack-operator can be exposed this way: keystone, placement, nova (API and novncproxy), glance, cinder, neutron, heat (api and cfnapi), swift, barbican, octavia, manila, ironic (api and inspector), designate, telemetry (aodh, prometheus, alertmanager), watcher and horizon. Whether the custom hostname and the `endpointURL` port are fully honored for a service depends on the version of that service's operator, see the release notes.

Service specific notes:

* horizon and the novncproxy have a different override layout, see [Services with a different layout](#services-with-a-different-layout).
* watcher (and cloudkitty, cyborg, prometheus, alertmanager) have no RHOSP 17.1 port; they keep their native port unless `endpointURL` sets one.
* several public Glance APIs can share one VIP only with distinct ports.
* the novncproxy of several cells needs a different port or IP per cell.

## Network and firewall requirements

* Clients must reach the **LoadBalancer IP on the front port** (the port of the `endpointURL`, not the port of the pod) on the network of the MetalLB pool. Open that port in the firewalls between the clients and that network.
* The MetalLB pool and its `L2Advertisement` must cover the network the clients are on; the IP must be unused elsewhere (see [Networking](networking.md)).
* Pods and data plane nodes that use the public endpoint need the same route to the LoadBalancer IP.
* Keep the internal LoadBalancer IPs unreachable from the public network with routing and firewall rules. This only controls who can reach the internal IPs; it does **not** restrict which virtual host is served on the public IP. Both endpoints are served by the same httpd and port and selected by the TLS server name (SNI), not by the IP the client connected to: a client that sends the internal name to the public IP reaches the internal virtual host, and a client without SNI always gets the internal virtual host. Routing and firewall rules cannot prevent this. Both virtual hosts proxy to the same service application and differ only in `ServerName` and certificate, so reaching the internal virtual host through the public IP gives no access beyond what the API's own authentication allows, and that authentication is what protects the API.
* Services on one shared IP need distinct front ports (see [Several public services on one IP](#several-public-services-on-one-ip)).

## Procedure: new deployment

1. Per public endpoint choose the **IP** (from the MetalLB pool), the **hostname** and the **port**. For shared IPs choose distinct ports.
2. Register the hostnames in DNS, see [DNS](#dns-registering-the-hostnames).
3. Decide on the certificate: default (operator-issued), custom issuer, or your own secret, see [Certificates](#certificates). Prepare the secret and the CA bundle if needed.
4. In the `OpenStackControlPlane` set the public Service to `type: LoadBalancer` with the MetalLB annotations and set `endpointURL` (hostname and port), plus `tls.api.public.secretName` for your own certificate.
5. Apply and wait for the control plane to be ready.
6. Verify, see [Verification](#verification): the Service has the IP and front port, the catalog shows the `endpointURL`, and the certificate presented for the hostname is the expected one and is trusted.

## Procedure: adoption from RHOSP 17.1

Goal: keep the VIP, hostnames, TLS ports and certificate of the 17.1 public endpoints.

1. Collect from the 17.1 deployment per service: the public VIP, the hostname clients use, the TLS port (the director ports in the [table above](#several-public-services-on-one-ip) unless changed), and the certificate, key and CA.
2. Free the VIP on the 17.1 side and provide it as MetalLB address (`loadBalancerIPs`) on the external network. Moving the VIP, and the firewall and routing changes, are manual and not covered here.
3. DNS: the existing records for the hostnames keep pointing to the VIP; check that the cluster upstream resolver and the dnsmasq upstream resolve them, see [DNS](#dns-registering-the-hostnames).
4. Create the secret with the existing certificate per endpoint (`tls.crt`, `tls.key`, `ca.crt`) and add the CA to `caBundleSecretName`. The certificate has to contain the hostname as SAN.
5. For each service configure the public Service as `LoadBalancer` on the VIP, `endpointURL: https://<hostname>:<17.1 port>` and `tls.api.public.secretName`. Services sharing the VIP keep working side by side as their 17.1 ports differ. To use another port than the 17.1 one, set it in the `endpointURL`.
6. Verify each service as described in [Verification](#verification) and switch clients over. Existing client configuration (URL, port, trusted CA) then needs no change.

## Verification

```bash
# Service type, IP and front port
oc get svc keystone-public -n openstack
# the catalog URL is exactly the endpointURL
oc rsh openstackclient openstack endpoint list --service identity --interface public
# the certificate presented for the hostname, via the LB IP
openssl s_client -connect 192.168.111.51:13000 -servername keystone.example.com \
  </dev/null 2>/dev/null | openssl x509 -noout -subject -ext subjectAltName -issuer
# name resolution from a pod
oc rsh openstackclient getent hosts keystone.example.com
# the certificate is trusted by the CA you expect
openssl s_client -connect 192.168.111.51:13000 -servername keystone.example.com \
  -CAfile company-ca.crt </dev/null 2>&1 | grep 'Verify return code'
# custom certificate state
oc get openstackcontrolplane -n openstack -o yaml | grep -A3 CustomTLSReady
```

## Invalid or conflicting configuration

Admission rejects these `endpointURL`/override combinations (the error names the field):

| Configuration | Fix |
|---|---|
| `endpointURL` without scheme or host, a non-FQDN host (`https://keystone:13000`) or an IP address | use a fully qualified hostname, `https://<fqdn>[:port]` |
| `override.service.public.spec.ports[].port` differs from the port of the `endpointURL` | remove the `ports` override, the `endpointURL` is enough |
| `override.service.public.spec.ports[].name` does not match the name of the service's port | use a name listed in the error, or remove the override |

The checks run in the webhook of each service operator, so a service operator without them accepts the configuration: the certificate then falls back to the in-cluster names (logged) and the port is not followed.

Not rejected, but a common mistake: `endpointURL` on an endpoint whose Service is still `ClusterIP` is ignored, the Route is created and the catalog shows the Route hostname. Set `type: LoadBalancer`.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Catalog shows `*.apps.<cluster>` instead of the custom URL | Public Service is still `ClusterIP`; Route created. Set `type: LoadBalancer`. |
| TLS handshake returns the *internal* certificate or a hostname mismatch | The service operator does not yet use `endpointURL` as vhost `ServerName`, or the client connects without SNI/with a different name than `endpointURL`. |
| `Verify return code` is not 0 / client says unknown CA | The CA of the certificate is not in the client's trust store, or not in `caBundleSecretName` for OpenStack services and EDPM nodes. |
| Hostname mismatch, certificate lacks the custom name | Bring-your-own certificate has no SAN for the host; or issuance is not finished, check `oc get certificate -n openstack`. |
| Connection refused / timeout on the custom port | Listener port differs from the URL (old service operator), or no route/firewall to the LB IP; check `oc get svc`. |
| Name does not resolve (from a pod, a node or a client) | Hostname not registered in all resolvers, see [DNS](#dns-registering-the-hostnames). |
| Two public Services stay `<pending>` on one IP | Same IP and same port; use explicit, distinct ports in the `endpointURL`s. |
| `CustomTLSReadyCondition` is False | The secret from `tls.api.*.secretName` is missing (retried every 10s) or invalid. |

## Limits

* Public endpoints only. A custom hostname for the **internal** endpoint is not supported.
* Virtual hosts are selected by the TLS server name (SNI), not by IP address. A client that sends no SNI is served the internal virtual host, and network separation does not change that, see [Network and firewall requirements](#network-and-firewall-requirements).
* Migration of an existing RHOSP 17.1 VIP to MetalLB (moving the IP, firewall and DNS changes) is a manual step outside of this document.
