# Events
This doc describes how events should be used in operators managing a podified controlplane (**NOT** osp-director-operator).

## General information
Description of events https://github.com/kubernetes/community/blob/master/contributors/devel/sig-instrumentation/event-style-guide.md#when-to-emit-an-event

## Events in openstack-k8s-operators podified controlplane operators
* Most of the events will be handled by lib-common, it will emit events when the owned resources state changes

## Example
```shell
oc get events --sort-by='.lastTimestamp' | grep openstack
34m         Normal    ServiceAccountCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   service account openstack-edpm-ipam created
34m         Normal    RoleBindingCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   rolebinding openstack-edpm-ipam created
34m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret dataplanenodeset-openstack-edpm-ipam created
34m         Warning   InputError                 openstackdataplanedeployment/edpm-deployment-ipam                               Secret "cert-ovn-edpm-compute-0" not found
34m         Normal    CertificateCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   certificate ovn-edpm-compute-0 created
34m         Warning   InputError                 openstackdataplanedeployment/edpm-deployment-ipam                               Secret "cert-ovn-edpm-compute-1" not found
34m         Warning   DataplaneDeploymentError            openstackdataplanenodeset/openstack-edpm-ipam                                   check deploymentStatuses for more details
34m         Normal    CertificateCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   certificate ovn-edpm-compute-1 created
34m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret openstack-edpm-ipam-ovn-certs-0 created
34m         Normal    NodeSetReady                 openstackdataplanenodeset/openstack-edpm-ipam                                   Setup complete
34m         Normal    CertificateCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   certificate neutron-metadata-edpm-compute-0 created
34m         Normal    NodeSetReady                 openstackdataplanenodeset/openstack-edpm-ipam                                   Setup complete
34m         Warning   NodeSetError                 openstackdataplanedeployment/edpm-deployment-ipam                               Secret "cert-neutron-metadata-edpm-compute-0" not found
34m         Normal    CertificateCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   certificate neutron-metadata-edpm-compute-1 created
34m         Warning   NodesetError                 openstackdataplanedeployment/edpm-deployment-ipam                               Secret "cert-neutron-metadata-edpm-compute-1" not found
34m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret openstack-edpm-ipam-neutron-metadata-certs-0 created
34m         Normal    CertificateCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   certificate libvirt-edpm-compute-1 created
34m         Warning   NodesetError                 openstackdataplanedeployment/edpm-deployment-ipam                               Secret "cert-libvirt-edpm-compute-1" not found
34m         Warning   NodesetError                 openstackdataplanedeployment/edpm-deployment-ipam                               Secret "cert-libvirt-edpm-compute-0" not found
34m         Normal    CertificateCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   certificate libvirt-edpm-compute-0 created
34m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret openstack-edpm-ipam-libvirt-certs-0 created
34m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   (combined from similar events): secret openstack-edpm-ipam-telemetry-certs-0 created
29m         Normal    ServiceAccountCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   service account openstack-edpm-ipam created
29m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret dataplanenodeset-openstack-edpm-ipam created
29m         Normal    RoleBindingCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   rolebinding openstack-edpm-ipam created
29m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret openstack-edpm-ipam-neutron-metadata-certs-0 created
29m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret openstack-edpm-ipam-ovn-certs-0 created
29m         Normal    NodeSetReady                 openstackdataplanenodeset/openstack-edpm-ipam                                   Setup complete
29m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret openstack-edpm-ipam-libvirt-certs-0 created
29m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret openstack-edpm-ipam-telemetry-certs-0 created
29m         Normal    SecretCreated                    openstackdataplanenodeset/openstack-edpm-ipam                                   secret openstack-edpm-ipam-nova-custom-certs-0 created
4m41s       Normal    NodeSetReady                 openstackdataplanenodeset/openstack-edpm-ipam                                   Setup complete
4m11s       Normal    DataplaneDeployment                 openstackdataplanedeployment/edpm-deployment-ipam                               Setup complete
```
