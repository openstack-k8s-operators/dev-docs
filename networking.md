# Networking

## Architecture

![Network diagram](images/network_diagram.jpg)

### Ingress

#### Public endpoints
Per default public endpoints get created as routes.

The OCP route hostname gets registered as the keystone public endpoint for the service.
```bash
+--------------+-----------+------------------------------------------------------------------+
| Service Name | Interface | URL                                                              |
+--------------+-----------+------------------------------------------------------------------+
| <service     | public    | http://<service>-public-openstack.apps.ostest.test.metalkube.org |
+--------------+-----------+------------------------------------------------------------------+
```

#### VIP endpoints
MetalLB service VIP (currently - in layer 2 mode) used for endpoints on networks which compute nodes need to be able to access. The connection is terminated at one node at the MetalLB VIP then forwarded to all service pods via the networking rules the kube-proxy has set.

The OCP LoadBalancer type service name gets registered as the keystone endpoint for the service. Controlplane services running on OCP will resolve those to the ClusterIP of the service, the Compute nodes have to resolve it to the External-IP of the LoadBalancer.
```bash
+--------------+-----------+---------------------------------------------------------------+
| Service Name | Interface | URL                                                           |
+--------------+-----------+---------------------------------------------------------------+
| <service>    | internal  | http://<service>-internal.openstack.svc:9292                  |
+--------------+-----------+---------------------------------------------------------------+
```

#### Individual endpoints
For services like OVS access via static IP on the isolated network is required.

This is right now a TODO and targeted when the IPAM solution, which is also required for EDPM nodes, has been finalized. Right now when the OVS pods restart, they'll likely get different IPs.

The goal is for those pods to use static IPAM plugin and pass the pod IP via the annotation.
```yaml
  annotations:
    k8s.v1.cni.cncf.io/networks: '[{"name": "internalapi", "namespace": "openstack", "ips": ["172.17.0.20/24"]}]'
```

### Egress
For outgoing isolated network traffic, pods be attached to additional networks using multus. The operators adds the `k8s.v1.cni.cncf.io/networks` annotation that references the list of network-attachment-definitions to which the deployment pods should be attached:


```yaml
annotations:
  k8s.v1.cni.cncf.io/networks: '[{"name": "internalapi", "namespace": "openstack"},{"name": "storage", "namespace": "openstack"}]'
```

## Env Pre-requisite
The NMstate and MetalLB operator need to be deployed and the OCP environment must support Multis CNI

### NMstate operator
The NMstate operator is performing state-driven network configuration across the cluster nodes with NMState. Further details can be found at:

* [About the Kubernetes NMState Operator](https://docs.openshift.com/container-platform/4.12/networking/k8s_nmstate/k8s-nmstate-about-the-k8s-nmstate-operator.html)
* [The Kubernetes NMState project](https://nmstate.io/kubernetes-nmstate/)

### MetalLB operator
The MetalLB operator provides a load-balancer implementation for bare metal clusters which brings user traffic into the clusters network. Currently only tested using MetalLB layer 2 mode. Further details can be found at:

* [Upstream Metallb documentation](https://metallb.universe.tf/)
* [About MetalLB and the MetalLB Operator](https://docs.openshift.com/container-platform/4.12/networking/metallb/about-metallb.html)

### Multus CNI
Multus CNI enables attaching multiple network interfaces to pods. For each of the networks, a network-attachment-definition defines how the network gets attached to the pod. Further details can be found at:

* [Understanding multiple networks](https://docs.openshift.com/container-platform/4.12/networking/multiple_networks/understanding-multiple-networks.html)
* [Using the Multus CNI in OpenShift](https://cloud.redhat.com/blog/using-the-multus-cni-in-openshift)
* [macvlan plugin](https://www.cni.dev/plugins/current/main/macvlan/)
* [whereabouts IPAM CNI plugin - Extended configuration](https://github.com/k8snetworkplumbingwg/whereabouts/blob/master/doc/extended-configuration.md)

## Steps to configure a controlplane service internal endpoints on an isolated network

### Networks used in this doc

| network name | CIDR | VLAN | Multus CNI | MetalLB IPAddresspool |
| --- | --- | --- | --- |--- |
| ctlplane | 192.168.122.0/24 | - | - | - |
| internalapi | 172.17.0.0/24 | 20 | 172.17.0.30-172.17.0.70 | 172.17.0.80-172.17.0.90 |
| storage | 172.18.0.0/24 | 21 | 172.18.0.30-172.18.0.70 | - |
| tenant | 172.19.0.0/24 | 22 | 172.19.0.30-172.19.0.70 | - |

### Connect OCP workers to the isolated networks
Worker nodes need to be connected to the required isolated networks. `NodeNetworkConfigurationPolicy` (`nncp`) resources can be used to apply the desired state via the NMstate operator to the worker nodes.

This `nncp` will configure the worker nodes interface `enp6s0` for using VLANs for network isolation:
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: osp
spec:
  desiredState:
    interfaces:
    - description: internalapi vlan interface
      ipv4:
        enabled: false
      ipv6:
        enabled: false
      name: enp6s0.20
      state: up
      type: vlan
      vlan:
        base-iface: enp6s0
        id: 20
    - description: storage vlan interface
      ipv4:
        enabled: false
      ipv6:
        enabled: false
      name: enp6s0.21
      state: up
      type: vlan
      vlan:
        base-iface: enp6s0
        id: 21
    - description: tenant vlan interface
      ipv4:
        enabled: false
      ipv6:
        enabled: false
      name: enp6s0.22
      state: up
      type: vlan
      vlan:
        base-iface: enp6s0
        id: 22
    - description: Configuring enp6s0
      ipv4:
        enabled: false
      ipv6:
        enabled: false
      mtu: 1500
      name: enp6s0
      state: up
      type: ethernet
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

### Network attachment definitions
For each isolated network, to which a service deployment pod should be attached, a `network-attachment-definition` (`nad`) need to be created.

This example `nad` for the internalapi network of type `macvlan` use the previous created `enp6s0.20` interface and the `whereabouts` CNI IPAM plugin to assign IPs to the created pods from the range `.30` - `.70`.

* internalapi network-attachment-definition
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: internalapi
  namespace: openstack
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "internalapi",
      "type": "macvlan",
      "master": "enp6s0.20",
      "ipam": {
        "type": "whereabouts",
        "range": "172.17.0.0/24",
        "range_start": "172.17.0.30",
        "range_end": "172.17.0.70"
      }
    }
```
A storage network `nad` would look almost the same as the above example for the internalapi, except of referencing `enp6s0.21` as the `master` and the network used in the `whereabouts` CNI IPAM plugin.

* storage network-attachment-definition
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: storage
  namespace: openstack
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "storage",
      "type": "macvlan",
      "master": "enp6s0.21",
      "ipam": {
        "type": "whereabouts",
        "range": "172.18.0.0/24",
        "range_start": "172.18.0.30",
        "range_end": "172.18.0.70"
      }
    }
```

* [macvlan plugin](https://www.cni.dev/plugins/current/main/macvlan/)
* [whereabouts IPAM CNI plugin - Extended configuration](https://github.com/k8snetworkplumbingwg/whereabouts/blob/master/doc/extended-configuration.md)

### MetalLB IP address pool
On the isolated network where a service needs to expose its endpoint using MetalLB, an `IPAddressPool` needs to be specified. A list of IP address ranges need to be specified over which MetalLB has authority, in this example `.80` - `.90`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: internalapi
  namespace: metallb-system
spec:
  addresses:
  - 172.17.0.80-172.17.0.90
  autoAssign: true
  avoidBuggyIPs: false
```

For details on other parameters check [Configuring MetalLB address pools](https://docs.openshift.com/container-platform/4.12/networking/metallb/metallb-configure-address-pools.html)

### MetalLB L2 advertisement
A `L2Advertisement` resource needs to be created which defines how the VIPs get announced. In layer 2 mode, one node assumes the responsibility of advertising a service to the local network. In this example VIPs requested from the `internalapi` address pool get announced on the interface which is attached to the internal api VLAN:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - internalapi
  interfaces:
  - enp6s0.20
```

* [METALLB IN LAYER 2 MODE](https://metallb.universe.tf/concepts/layer2/)
* [Specify network interfaces that LB IP can be announced from](https://metallb.universe.tf/configuration/_advanced_l2_configuration/#specify-network-interfaces-that-lb-ip-can-be-announced-from)
* [About advertising for the IP address pools](https://docs.openshift.com/container-platform/4.12/networking/metallb/about-advertising-ipaddresspool.html)

### Configure exposing services to isolated networks
#### Service API component
Each API component of a service supports exposing their endpoint to an isolated network

```yaml
  externalEndpoints:
    description: ExternalEndpoints, expose a VIP using a pre-created IPAddressPool
    items:
      description: MetalLBConfig to configure the MetalLB loadbalancer service
      properties:
        endpoint:
          description: Endpoint, OpenStack endpoint this service maps to
          enum:
          - internal
          - public
          type: string
        ipAddressPool:
          description: IPAddressPool expose VIP via MetalLB on the IPAddressPool
          minLength: 1
          type: string
        loadBalancerIPs:
          description: LoadBalancerIPs, request given IPs from the pool
            if available. Using a list to allow dual stack (IPv4/IPv6) support
          items:
            type: string
          type: array
        sharedIP:
          default: true
          description: SharedIP if true, VIP/VIPs get shared with multiple services
          type: boolean
        sharedIPKey:
          default: ""
          description: SharedIPKey specifies the sharing key which gets
            set as the annotation on the LoadBalancer service. Services
            which share the same VIP must have the same SharedIPKey. Defaults
            to the IPAddressPool if SharedIP is true, but no SharedIPKey
            specified.
          type: string
      required:
      - endpoint
      - ipAddressPool
```

Example:
```yaml
  externalEndpoints:
  - endpoint: internal
    ipAddressPool: internalapi
    loadBalancerIPs:
    - 172.17.0.80
    sharedIP: true
    sharedIPKey: ""
```

#### RabbitMQ
RabbitMQ instances can be exposed to an isolated network using the `externalEndpoint` parameter in the `OpenStackControlPlane` CRD.
**Note** multiple RabbitMQ instances can not share the same VIP as they use the same port. If multiple RabbitMQ instances need to be exposed to the same network distinct ip address have to be used.

```yaml
  externalEndpoint:
    description: ExternalEndpoint, expose a VIP via MetalLB
      on the pre-created address pool
    properties:
      ipAddressPool:
        description: IPAddressPool expose VIP via MetalLB on
          the IPAddressPool
        minLength: 1
        type: string
      loadBalancerIPs:
        description: LoadBalancerIPs, request given IPs from
          the pool if available. Using a list to allow dual
          stack (IPv4/IPv6) support
        items:
          type: string
        type: array
      sharedIP:
        default: true
        description: SharedIP if true, VIP/VIPs get shared with
          multiple services
        type: boolean
      sharedIPKey:
        default: ""
        description: SharedIPKey specifies the sharing key which
          gets set as the annotation on the LoadBalancer service.
          Services which share the same VIP must have the same
          SharedIPKey. Defaults to the IPAddressPool if SharedIP
          is true, but no SharedIPKey specified.
        type: string
    required:
    - ipAddressPool
    type: object
```

Example exposing two RabbitMQ instances to the internal api network:
```yaml
  rabbitmq:
    templates:
      rabbitmq:
        externalEndpoint:
          loadBalancerIPs:
          - 172.17.0.85
          ipAddressPool: internalapi
          sharedIP: false
      rabbitmq-cell1:
        externalEndpoint:
          loadBalancerIPs:
          - 172.17.0.86
          ipAddressPool: internalapi
          sharedIP: false
```

### Example: expose GlanceAPI to an isolated network

The Glance spec can be used to configure Glance to register e.g. the internal endpoint to an isolated network.

The following represents an example of `Glance` resource that can be used to trigger the service deployment, and have the internal `GlanceAPI` endpoint registerd as a MetalLB service using the IPAddressPool `internalapi`, request to use the IP `172.17.0.80` as the VIP and the IP is shared (default) with other services.

```
apiVersion: glance.openstack.org/v1beta1
kind: Glance
metadata:
  name: glance
spec:
  ...
  glanceAPIInternal:
    ...
    externalEndpoints:
      - endpoint: internal
        ipAddressPool: internalapi
        loadBalancerIPs:
        - "172.17.0.80"
    ...
...
```

The internal glance endpoint gets registered with its service name. This service name needs to resolve to the `LoadBalancerIP` on the isolated network either by DNS or via /etc/hosts:

```
# openstack endpoint list -c 'Service Name' -c Interface -c URL --service glance
+--------------+-----------+---------------------------------------------------------------+
| Service Name | Interface | URL                                                           |
+--------------+-----------+---------------------------------------------------------------+
| glance       | internal  | http://glance-internal.openstack.svc:9292                     |
| glance       | public    | http://glance-public-openstack.apps.ostest.test.metalkube.org |
+--------------+-----------+---------------------------------------------------------------+
```
## Attaching service pods to additional networks
Each service supports attaching their deployments to isolated networks by referencing network-attachment-definition names using the `networkAttachments` parameter.

```yaml
  networkAttachents:
  - internalapi
  - storage
```

### Example: configure Glance with additional networks
The Glance spec can be used to configure the GlanceAPI pods to be attached to additional networks e.g. connect to a Ceph RBD server on a dedicated storage network.

The following represents an example of Glance resource that can be used to trigger the service deployment, and have the service pods attached to the storage network using a [storage network-attachment-definition](###-network-attachment-definitions) .

```
apiVersion: glance.openstack.org/v1beta1
kind: Glance
metadata:
  name: glance
spec:
  ...
  glanceAPIInternal:
    ...
    networkAttachents:
    - storage
  glanceAPIExternal:
    ...
    networkAttachents:
    - storage
...
```

When the service is up and running, it will now have an additional nic configured for the storage network:

```
# oc rsh glance-external-api-dfb69b98d-mbw42
Defaulted container "glance-api" out of: glance-api, init (init)
sh-5.1# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if298: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 0a:58:0a:82:01:18 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.130.1.24/23 brd 10.130.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::4cf2:a3ff:feb0:932/64 scope link
       valid_lft forever preferred_lft forever
4: net1@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether a2:f1:3b:12:fd:be brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.52/24 brd 172.18.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::a0f1:3bff:fe12:fdbe/64 scope link
       valid_lft forever preferred_lft forever
```

## Example: configure OpenStackControlPlane with isolated networks
Examples on how to use the `OpenStackControlPlane` CRD can be found at
* https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation.yaml
* https://github.com/openstack-k8s-operators/openstack-operator/blob/main/config/samples/core_v1beta1_openstackcontrolplane_network_isolation_ceph.yaml

## DNS
The the infra-operator privides CRDs to setup and manage a dnsmasq instances for DNS.

The `DNSMasq` CRD allows creating a dnsmasq DNS server instance and expose the service via `ExternalEndpoints` as with the API endpoints via MetalLB to systems on isolated networks.

Using `Spec.Options` the instance can be customized from the default config started. E.g. adding a DNS server to forward requests to which are not resolvable vie the local instances can be done like:

```yaml
spec:
...
  options:
  - key: server
    values:
    - 192.168.122.1
```

Information about hosts to resolve is being added via configmaps which have a label with `dnsmasqhosts: dnsdata` (can be customized via `DNSDataLabelSelectorValue`) set.

The `DNSData` CRD allows to register hosts, which should be resolved in the dnsmasq instance. The CRD creates the required configmap with the label as
described above.

```yaml
apiVersion: network.openstack.org/v1beta1
kind: DNSData
metadata:
  name: somehosts
  namespace: openstack
spec:
  dnsDataLabelSelectorValue: dnsdata
  hosts:
  - hostnames:
    - edpm-compute-0.ctlplane
    ip: 192.168.122.100
  - hostnames:
    - edpm-compute-0.internalapi
    ip: 172.17.0.100
  - hostnames:
    - edpm-compute-0.storge
    ip: 172.18.0.100
  - hostnames:
    - edpm-compute-0.tenant
    ip: 172.19.0.100
  - hostnames:
    - edpm-compute-1.ctlplane
    ip: 192.168.122.101
  - hostnames:
    - edpm-compute-1.internalapi
    ip: 172.17.0.101
  - hostnames:
    - edpm-compute-1.storge
    ip: 172.18.0.101
  - hostnames:
    - edpm-compute-1.tenant
    ip: 172.19.0.101
```

Additionally there is a CRD-less `service_controller`. It watches for `LoadBalancer` services in the same namespace of a `DNSMasq` instance and creates a `DNSData` CR to auto register service endpoints which have an annotation set with  `"dnsmasq.network.openstack.org/hostname": <hostname>`. This annotation gets auto added to the `LoadBalancer` services exposed using lib-common `https://github.com/openstack-k8s-operators/lib-common/pull/258`.

## IPAM
The the infra-operator privides CRDs for IPAM.

The `NetConfig` CRD allows to describe the overall networks used for the environment.

* a `network` described by `name`, `mtu` (default 1500), `dnsDomain` and one or multiple subnets.
* a `subnet` described by: `name`, `cidr`, `vlan` (id), `gateway`, `routes`, `excludeAddresses` and `allocationRanges`. One or multiple `allocationRanges` can be defined, each with a `start` and `end` address. `routes` are optional, but one or multiple can be defined. Each route defining a `destination` network and `nexthop` router address. Using `excludeAddresses` individual IP addresses from within the `cidr` can be excluded to be assigned (dynamic or static). A `dnsDomain` can also be set on the subnet. This will have precedence over the `dnsDomain` from the network on the IPSet status.

Example `NetConfig`:
```yaml
apiVersion: network.openstack.org/v1beta1
kind: NetConfig
metadata:
  name: netconfig
spec:
  networks:
  - name: CtlPlane
    dnsDomain: ctlplane.example.com
    subnets:
    - name: subnet1
      allocationRanges:
      - end: 192.168.122.120
        start: 192.168.122.100
      - end: 192.168.122.200
        start: 192.168.122.150
      cidr: 192.168.122.0/24
      gateway: 192.168.122.1
  - name: InternalApi
    dnsDomain: internalapi.example.com
    subnets:
    - name: subnet1
      allocationRanges:
      - end: 172.17.0.250
        start: 172.17.0.10
      excludeAddresses:
      - 172.17.0.10
      - 172.17.0.12
      cidr: 172.17.0.0/24
      vlan: 20
  - name: External
    dnsDomain: external.example.com
    subnets:
    - name: subnet1
      allocationRanges:
      - end: 10.0.0.250
        start: 10.0.0.10
      cidr: 10.0.0.0/24
      gateway: 10.0.0.1
      routes:
      - destination: 10.0.1.0/24
        nexthop: 10.0.0.100
  - name: Storage
    dnsDomain: storage.example.com
    subnets:
    - name: subnet1
      allocationRanges:
      - end: 172.18.0.250
        start: 172.18.0.10
      cidr: 172.18.0.0/24
      vlan: 30
  - name: StorageMgmt
    dnsDomain: storagemgmt.example.com
    subnets:
    - name: subnet1
      allocationRanges:
      - end: 172.19.0.250
        start: 172.19.0.10
      cidr: 172.19.0.0/24
      vlan: 40
  - name: Tenant
    mtu: 9000
    subnets:
    - name: subnet1
      allocationRanges:
      - end: 172.20.0.250
        start: 172.20.0.10
      cidr: 172.20.0.0/24
      vlan: 50
```

The `IPSet` CRD can be used by a requestor to register IP addresses on one or more networks.

The request is a list of networks with `name` and `subnetName`, optional `defaultRoute` for one subnet in total. Using the `fixedIP` a specific IP can be requested, if not already assigned. The `fixedIP` don't have to be from within an `allocationRange`.

The `immutable` parameter prevents updates to the IPSet spec via the webhook. This is a level of security to prevent changes to the object. If changes are required an additional step is required to set the `immutable` parameter to `false`, which indicates "yes I really know what I am doing".

The status if the `IPSet` will have all details about the reserved IPs and network details to be used by the requestor. The `IPSet` can be used by the requestor if the overall `Ready` condition is `True`.

Example to request IP reservation with requesting a `fixedIP` on the `InternalApi` network `subnet1`:
```yaml
kind: IPSet
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"network.openstack.org/v1beta1","kind":"IPSet","metadata":{"annotations":{},"name":"edpm-compute-0","namespace":"openstack"},"spec":{"networks":[{"defaultRoute":true,"name":"CtlPlane","subnetName":"subnet1"},{"fixedIP":"172.17.0.150","name":"InternalApi","subnetName":"subnet1"},{"name":"Storage","subnetName":"subnet1"},{"name":"Tenant","subnetName":"subnet1"},{"name":"External","subnetName":"subnet1"}]}}
  creationTimestamp: "2023-05-25T07:27:31Z"
  finalizers:
  - IPSet
  generation: 1
  name: edpm-compute-0
  namespace: openstack
  resourceVersion: "4916903"
  uid: 3901b2f3-f724-48fb-a906-7b46e9a5b19a
spec:
  immutable: true
  networks:
  - defaultRoute: true
    name: CtlPlane
    subnetName: subnet1
  - fixedIP: 172.17.0.150
    name: InternalApi
    subnetName: subnet1
  - name: Storage
    subnetName: subnet1
  - name: Tenant
    subnetName: subnet1
  - name: External
    subnetName: subnet1
status:
  conditions:
  - lastTransitionTime: "2023-05-25T07:27:31Z"
    message: Setup complete
    reason: Ready
    status: "True"
    type: Ready
  - lastTransitionTime: "2023-05-25T07:27:31Z"
    message: Input data complete
    reason: Ready
    status: "True"
    type: InputReady
  - lastTransitionTime: "2023-05-25T07:27:31Z"
    message: Reservation successful
    reason: Ready
    status: "True"
    type: ReservationReady
  reservations:
  - address: 192.168.122.100
    cidr: 192.168.122.0/24
    dnsDomain: ctlplane.example.com
    gateway: 192.168.122.1
    mtu: 1500
    network: CtlPlane
    subnet: subnet1
  - address: 10.0.0.10
    cidr: 10.0.0.0/24
    dnsDomain: external.example.com
    gateway: 10.0.0.1
    mtu: 1500
    network: External
    routes:
    - destination: 10.0.1.0/24
      nexthop: 10.0.0.100
    subnet: subnet1
  - address: 172.17.0.150
    cidr: 172.17.0.0/24
    dnsDomain: internalapi.example.com
    mtu: 1500
    network: InternalApi
    subnet: subnet1
    vlan: 20
  - address: 172.18.0.10
    cidr: 172.18.0.0/24
    dnsDomain: storage.example.com
    mtu: 1500
    network: Storage
    subnet: subnet1
    vlan: 30
  - address: 172.20.0.10
    cidr: 172.20.0.0/24
    dnsDomain: tenant.example.com
    mtu: 1500
    network: Tenant
    subnet: subnet1
    vlan: 50
```

The reservation is also storted via the `Reservation` CRD, but it is intended for "internal" use only.
