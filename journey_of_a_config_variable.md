# The journey of a config variable in openstack-k8s-operators

Our goal is to pass extra configuration from the CI job definition to the
nova-compute process running on the EDPM compute node. E.g. we need to pass
the following
```ini=
[libvirt]
cpu_mode = custom
cpu_models = Nehalem
```
to nova-compute in CI to allow live migration between slightly different
compute node models.

The config goes through the following journey:
1. The snippet starts its life in the
   [.zuul.yaml defining the CI job](https://github.com/openstack-k8s-operators/nova-operator/blob/03ba9b30065bc410df99a45ed3aff3bfb28c43f3/.zuul.yaml#L182-L185)
2. That config snippet is copied into a .conf file.
   [by the ansible code in ci_framework](https://github.com/openstack-k8s-operators/ci-framework/blob/28e8d32147d0c71f4060c3da6fc38090b35d6aa0/roles/edpm_deploy/tasks/main.yml#L41-L44)
3. Then the
   [filepath is passed](https://github.com/openstack-k8s-operators/ci-framework/blob/28e8d32147d0c71f4060c3da6fc38090b35d6aa0/roles/edpm_deploy/tasks/main.yml#L46-L53)
   to the
   [`edpm_deploy_prep` makefile target of install_yamls](https://github.com/openstack-k8s-operators/install_yamls/blob/80eac86dc1a86147f41bf41aeb754309876fbee0/Makefile#L696-L705)
   via the`DATAPLANE_EXTRA_NOVA_CONFIG_FILE` ENV variable.
4. The install_yamls target uses kustomize to
   [generate a ConfigMap](https://github.com/openstack-k8s-operators/install_yamls/blob/80eac86dc1a86147f41bf41aeb754309876fbee0/scripts/gen-edpm-kustomize.sh#L210-L218)
   that will store the config snippet in k8s.
5. The ConfigMap gets created in k8s during the early phases of the EDPM
   deployment by the
   [joint effort of install_yamls and ci_framework](https://github.com/openstack-k8s-operators/ci-framework/blob/b66ddaf89f3034870e55955786eb5eb7598017ff/ci_framework/roles/edpm_deploy/tasks/main.yml#L47-L115).
6. A new [`OpenStackDataPlaneService` CR](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/docs/openstack_dataplaneservice.md)
   referencing the
   [`osp.edpm.nova` ansible playbook](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/playbooks/nova.yml)
   is [created](https://github.com/openstack-k8s-operators/install_yamls/blob/80eac86dc1a86147f41bf41aeb754309876fbee0/scripts/gen-edpm-kustomize.sh#L195-L208)
   from the [default nova service sample](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/config/services/dataplane_v1beta1_openstackdataplaneservice_nova.yaml)
   to reference the new ConfigMap. We need to do this as the default service
   CR is not modifiable.
7. The default [`OpenStackDataPlaneNodeSet` CR](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/docs/openstack_dataplanenodeset.md)
   is [changed](https://github.com/openstack-k8s-operators/install_yamls/blob/80eac86dc1a86147f41bf41aeb754309876fbee0/scripts/gen-edpm-kustomize.sh#L232)
   to use the new `OpenStackDataPlaneService` CR instead of the default one.
8. When the
   [`OpenStackDataPlaneDeployment` CR](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/docs/openstack_dataplanedeployment.md)
   is [created](https://github.com/openstack-k8s-operators/ci-framework/blob/28e8d32147d0c71f4060c3da6fc38090b35d6aa0/roles/edpm_deploy/tasks/main.yml#L82-L150)
   the [dataplane-operator translates](https://github.com/openstack-k8s-operators/dataplane-operator/blob/8d8a1051288a065e8fec16e07c907dc08ebbb4d8/controllers/openstackdataplanedeployment_controller.go#L225-L228)
   the `OpenStackDataPlaneService` CR to
   an [`OpenStackAnsibleEE` CR](https://github.com/openstack-k8s-operators/openstack-ansibleee-operator/blob/main/docs/openstack_ansibleee.md)
   and [ensures](https://github.com/openstack-k8s-operators/dataplane-operator/blob/b8c94dc3631cef634f29760e07b938595a630572/pkg/deployment/deployment.go#L200)
   that the ConfigMap is translated to an item in the `extraVolumes` field of
   the `OpenStackAnsibleEE` CR.
9. The [openstack-ansibleee-operator creates a k8s Job](https://github.com/openstack-k8s-operators/openstack-ansibleee-operator/blob/a0ea58b83c3acc84d450723dda4d356c2555f781/controllers/openstack_ansibleee_controller.go#L204-L207)
   from the `OpenStackAnsibleEE` CR and instructs k8s to
   [mount the ConfigMap to the Pod](https://github.com/openstack-k8s-operators/openstack-ansibleee-operator/blob/a0ea58b83c3acc84d450723dda4d356c2555f781/controllers/openstack_ansibleee_controller.go#L389)
   created from the Job.
10. The `osp.edpm.nova` ansible playbook runs the
    [`edpm_nova` ansible role](https://github.com/openstack-k8s-operators/edpm-ansible/tree/main/roles/edpm_nova/tasks)
    in the k8s Pod where our ConfigMap content is mounted as a file now.
11. The `edpm_nova` role [copies every config file](https://github.com/openstack-k8s-operators/edpm-ansible/blob/e53be776f54e6ea1f459e8f5e36af89539eeb47a/roles/edpm_nova/tasks/configure.yml#L81-L92)
    to the EDPM node (hypervisor) into `/var/lib/openstack/config/nova`
    directory then
    [starts the nova_compute container](https://github.com/openstack-k8s-operators/edpm-ansible/blob/e53be776f54e6ea1f459e8f5e36af89539eeb47a/roles/edpm_nova/tasks/install.yml#L3-L26)
    with podman mounting the `/var/lib/openstack/config/nova` directory
    into it.
12. Kolla in the nova_compute container copies the config files to
    `/etc/nova/nova.conf.d/` directory and starts the nova-compute binary and
    the config snippet gets loaded by `oslo.config` and the magic happens.
