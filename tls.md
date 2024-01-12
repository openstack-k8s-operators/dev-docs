# TLS

TLS is enabled per default

## Endpoint Termination overview
* Per default public endpoints terminate at the route
  * The openstack-operator creates those routes for each of the services
  * but using the `serviceOverride` it is possible to use a `LoadBalancer` service for public endpoints, too.
* Internal endpoints terminate at the service, either `ClusterIP` or `LoadBalancer` type service (e.g. MetalLB)

More details on networking check [Networking](./networking.md)

## TLS config

Per default two CAs, managed by cert-manager, will be used for public and internal endpoints. There might be services which will use their own CA.

### General TLS control
General TLS settings be changed using the tls section of the `OpenStackControlPlane`:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: myctlplane
spec:
  tls:
    endpoint:
      internal:
        enabled: false
      public:
        enabled: true
    caBundleSecretName: myAdditionalCACerts
```

Using the `caBundleSecretName` parameter a secret can be referenced containing any additional CA certificates, which should be added to the CA bundle.

### TLS public endpoints

Per default certificates get requested/created for each public endpoint. In addition the user can
* create a secret per service to provide a custom certificate (public/corp CA). This secret must provide `tls.key`, `tls.crt` and `ca.crt`. The information is then used to be set on the `Route` for this public endpoint or in future if a `LoadBalancer` service is used to terminated the endpoint at the pod level.
* An alternative would be to direct use the `RouteOverride` to set the cert, key and cacert for the endpoint

Example using route override:
```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: myctlplane
spec:
  ...
  keystone:
    apiOverride:
      route:
        spec:
          tls:
            termination: edge
            certificate: |-
              -----BEGIN CERTIFICATE-----
              -----END CERTIFICATE-----
            key: |-
              -----BEGIN RSA PRIVATE KEY----
              -----END RSA PRIVATE KEY-----
            caCertificate: |-
              -----BEGIN CERTIFICATE-----
              -----END CERTIFICATE-----
            insecureEdgeTerminationPolicy: Redirect
```

### Combinded CA bundle

The `openstack-operator` creates a combindes CA bundle, stored in a secret named `combined-ca-bundle`. This bundle will have:
* all default CAs from the operator image itself (system CAs)
* the CAs created by the `openstack-operator`
* any 3rd party CA provided via the `tls.caBundleSecretName`
* CAs provided via the `apiOverride.tls.secretName`

Note: The CA probided via the `apiOverride.route` is right now *not* added to the CA bundle.

The secret also has a default label which can be used to query it:
```yaml
  labels:
    combined-ca-bundle: ""
```

Whenever `openstack-operator` reconciles it will check for expired or new CA certs to be added to the bundle. Not expired CAs will be kept in the secret if they are not yet expired, even if they got removed from the source it was originally taken from. This allows to rotate certs for a service using a new CA.

This bundle can be used to be direct mounted into a deployment to the environment wide CA location. The [lib-common common module tls package](https://github.com/openstack-k8s-operators/lib-common/tree/main/modules/common/tls) provides funtionality for this.

### TLS internal communication

Each service operator creates for its APIs one or multiple k8s services. If in the `OpenStackControlPlane` CR gets configured to have tls for internal endpoints enabled:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: myctlplane
spec:
  tls:
    endpoint:
      internal:
        enabled: true
      ...
```

Most of the deployments use k8s service as a single entry to access the service pods, but there might some which don't. In general TLS termination happens at the pod level.

#### Deployments using k8s services

The configuration of the service is reflecting the k8s service name, not the individual pod name. With this all pods in a scaling resource will share the same service certificate. 

For each of the k8s services the openstack-operator requests a certificate for the hostname <service>.<namespace>.svc using the internal and the public `CertManager` issuer. For each cert request, CertManger will create a secret holding a `tls.crt`, `tls.key` and `ca.crt`. Those secrets, certificate and the combined CA bundle described above, can be passed into the service operators CRD using the tls section.

Public/internal API services:
```yaml
  tls:
    api:
      # secret holding tls.crt and tls.key for the APIs internal k8s service
      internal:
        secretName: cert-internal-svc
      # secret holding tls.crt and tls.key for the APIs public k8s service
      public:
        secretName: cert-public-svc
    # secret holding the tls-ca-bundle.pem to be used as a deploymend env CA bundle
    caBundleSecretName: combined-ca-bundle
```

Single, not endpoint specific service:
```yaml
  tls:
    caBundleSecretName: combined-ca-bundle
    secretName: cert-nova-novncproxy-cell1-public-svc
```

As mentioned above, the CA bundle will be mounted as the global CA bundle for the deployment container. The service secrets will be used for the service providing the api. Depending on how the service gets started (using kolla or not), the cert and key get either mounted to a directory and using kolla to put into its final place, or direct mounted to where the service expects it.

For k8s service a virtual host gets configured using the service name with its corresponding certificate.

Like with other config secrets/config maps a hash gets used to identify if the certificate changed and the deployment pod has to be restarted. For this, the service operators index the named secret fields to be able to watch those if they change. If a change happenes which result in a hash change, the default action is to restart the deployment. There might be services which will handle this different.

If internal tls is enabled, using `spec.tls.endpoint.internal.enabled: true`, routes get configured with `spec.tls.termination: reencrypt` and CA added to `spec.tls.destinationCACertificate` to be able to validate the certificate of the k8s service.

#### Deployments not using k8s services

TBD when e.g. ovn got added
