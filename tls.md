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
    caSecretName: myAdditionalCACerts
```

Using the `caSecretName` parameter a secret can be referenced containing any additional CA certificates, which should be added to the CA bundle.

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
* any 3rd party CA provided via the `tls.caSecretName`
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

TBD
