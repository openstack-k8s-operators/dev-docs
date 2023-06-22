#  Services configuration / k8s operators

## Goal

Describe an approach that can be adopted across the operators to improve the
deployment aspect and the way secrets and config files are generated. The
solution proposed in this document, in the first place, is to give an operator
the ability to render and inject sensitive snippets within the service config
deployed in the OpenStack controlplane.

### Note:

The document is not supposed to cover the implementation details that might
converge or diverge according to a given operator, the number of deployed
services, and other potential requirements that are not the same across the
board.


## Proposed approach

The use cases are concentrated in two main aspects:

* Provide a common interface that can be used to build additional config
  snippets containing sensitive information using k8s `Secrets` instead of
  `ConfigMaps`

* Provide a common pattern to build regular service config:
  * they will be processed via golang templates and mounted in a Pod from a
    `ConfigMap` (which is what is currently happening)
  * part of them will be rendered in a `Secret` and avoid exposing sensitive
    information related to the deployment aspect


## Basic deployment template

Usually, the basic deployment config information is rendered in a `ConfigMap`.
However, the idea is to store sensitive information in a `Secret` that will be
mounted in the service Pod. For this reason, a given operator should be
responsible to process “on board” the information coming from the main
`osp_secret`, and generate a new `Secret` that matches to the deployment golang
template that will be used by other service components.

```
                                                    +------------------------------------------------------------------------------------+
1. The template is processed by the operator        | [database]                                                                         |
2. A new Secret is created and  mounted as a Volume | connection = mysql+pymysql://{{.DBUser}}:{{.DBPassword}}@{{.DBHost}}/{{.Database}} |
   by the Pod                                       |                                                                                    |
3. Kolla sync src / dest and applies the expected   | [keystone_authtoken]                                                               |
   ownership/permissions                            | password = {{.Password}}                                                           |
                                                    | ...                                                                                |
                     +----------------------------- | ...                                             (1)                                |
                     |                              | [service_user]                                                                     |
                     |                              | password = {{.Password}}                                                           |
                     |                              +------------------------------------------------------------------------------------+
                     |
+----------------------------------------+      +----------------------------------------------------------------------------------------+
| apiVersion: v1                   (2)   |      |  “config_files”: [        (3)                                                          |
| metadata:                              |      |                  ...                                                                   |
|   name: 01-<service>-deployment        |      |      {                                                                                 |
|   namespace: openstack                 | ===> |        "source": "/var/lib/config-data/deployment/01-<service>.conf",|
| stringData:                            |      |        "dest": "/etc/<service>/<service>.conf.d/01-<service>.conf",  |
|   01-<service>.conf: |                 |      |        "owner": "<service>",                                                           |
|        <data>                          |      |        "perm": "0600"                                                                  |
+----------------------------------------+      |      },                                                                                |
                                                +----------------------------------------------------------------------------------------+
```

As the diagram above depicts, a given operator is supposed to implement the
logic to build and reconcile the `Secret` when the template has been processed.
The `Secret`, mounted to the resulting service Pod, in the very last step of
this process can be synced by kolla to the destination directory: instead of
having a bash script (e.g. `init.sh`) doing the `chown` on the destination folder,
we'll rely (where possible) on kolla that allows setting ownership and
permissions, as well as copying optional files in case additional configuration
is provided.

## Render multiple secrets: “CustomServiceConfigSecrets” interface

Currently the `OpenStack` storage operators expose the
`customConfigServiceSecrets` parameter, which provides a mechanism for
specifying service configs via Secrets. Instead of specifying sensitive config
data directly in the `customServiceConfig`, a cloud admin can place sensitive
data in a `Secret`, and reference the secret by name in the service's
`customServiceConfigSecrets` as a list of `Secret` names, that will be
iterated without any form of sorting (the same order how they appear in the list
is used).

```go
for _, name := range instance.Spec.CustomServiceConfigSecrets {
  confSecret, _, err := oko_secret.GetSecret(ctx, helper, name, instance.Namespace)
    if err != nil {
      // Secret not found or unable to retrieve the secret, returning err
      return ctrlResult, err
    }
    confSecrets = append(confSecrets, *confSecret)
}
```

Such parameter is used to select and collect existing k8s Secrets (without
decoding any value in plaintext) and provide them as Volumes/VolumeMounts, that
are processed by the operator and mounted in the target Pod.

```
     +----------------------------+
     | customServiceConfigSecret: |
 +---|   - service-secret1        |--------------------
 |   |   - service-secret2        |                   |
 |   +----------------------------+                   |
 |         |                                   +-------------+
 |         |                                   | - snippet 1 |
 |  +------------------------------------+     | - snippet 2 | 
 |  | apiVersion: v1                     |     | - snippet 3 |
 |  | kind: Secret                       |     +-------------+
 |  | metadata:                          |            |
 |  |   name: service-secret1            |            |
 |  |   namespace: openstack             |            |
 |  | stringData:                        |  +----------------------------------+
 |  |   snippet1: |                      |  | 04-secret-<service>.conf         |
 |  |   [logger_root]                    |  |                                  |
 |  |   level=WARNING                    |  | [logger_root]                    |
 |  |   handlers=stdout                  |  | level=WARNING                    |
 |  |   snippet2: |                      |  | handlers=stdout                  |
 |  |   ##################               |  | ##################               |
 |  |   # Log Formatters #               |  | # Log Formatters #               |
 |  |   ##################               |  | ##################               |
 |  |   [formatter_normal]               |  | [formatter_normal]               |
 |  |   format=(%(name)s): s%(message)s  |  | format=(%(name)s): s%(message)s  |
 |  +------------------------------------+  | [formatters]                     |
 |                                          | keys=normal                      |
+-------------------------+                 +----------------------------------+
| apiVersion: v1          |                           |
| kind: Secret            |                           |
| metadata:               |                           |
|   name: service-secret2 |                           |
|   namespace: openstack  |        1. Mounted as Volume by Service Pod
| stringData:             |
|   snippet3: |           |        2. `kolla_set_configs && kolla_start`: sync
|  [formatters]           |           the resulting secret as done for the other
|  keys=normal            |           regular config files
+-------------------------+
```

An example of the `customServiceConfigSecret` usage can be found in Manila,
where this parameter has been used to test the
[NetApp backend](https://gist.github.com/gouthampacha/1b5681104ee066b5bd2c702b29376199)

### Note

No one prevents the cloud admin to hold multiple Secrets where each secret can
have many (even overlapping) oslo keys or config snippets. As stated earlier,
secrets are processed in the order they appear in the CR parameter, and the
concatenation of secrets containing duplicated sections can be problematic and
presents constraints, hence it's strongly recommended to provide the same
snippet or key only once. If multiple snippets are provided in a single Secret
referenced in `customServiceConfigSecrets`, a predictable iteration order
should be provided, hence the snippets are applied to the service config in the
lexicographic ordering of the keys of those snippets.

As a result of this strategy, the service presents a layout similar to the
following:

```
00-default.conf            => default configs generated by operator. This is stored in ConfigMap or Secret
01-deployment.conf         => default configs generated by operator, which contains credentials such as [database] connection.
                              This is stored in Secret
02-global.conf             => custom configs provided by users via top-level customServiceConfig. This is stored in ConfigMap or Secert
03-service.conf            => custom configs provided by users via service level customServiceConfig. This is stored in ConfigMap or Secret
04-secrets.conf            => custom configs provided by users via service level customServiceConfigSecrets, which contains credentials.
                              This is stored in Secret and would not be present if no secrets are provided
```


The service will pass the `--config-dir` parameter to point to the
`<service>.conf.d` directory where all the config files listed above are
rendered.

```
"command": "/usr/bin/<service> --config-dir /etc/<service>/<service>.conf.d"
```

If this strategy is not available, the `init container` that executes a start
script (e.g., `init.sh`) won't be removed, and the logic that generates the
layout mentioned above will be implemented in the init script until the missing
support is added in the upstream project.
Init containers are still required in some cases: for instance, if a given
service needs to be exposed via Multus on a particular network and the the IP
information is required for its config (and the mentioned IP information is
only available after the Pod is created and started), using an init container
will help addressing such scenario.

## Conclusion

The model described here allows to reach many goals:
- when possible, if a given service has no particular requirements, remove the
  `initContainer` and the related scripts that are no longer required to start
  the service: the `deployment Secret` is generated by the operator according
  to the parameters defined in the service CR and the data retrieved by the
  initial `osp_secret`

- `Kolla` is still used  to copy files from `src` to `dst`, as they are
  rendered and mounted accordingly with the right permissions in the
  destination directory (*)

- Operators’ controllers are able to parse many secrets referenced by the
  `CustomServiceConfigSecrets` parameter and merge them into a **single**
  `Secret` which is passed to the deployment and mounted to the `Pod` in the
  target directory (**)

Due the reasons mentioned above, `kolla` is still the target tool used to start
the service.

(\*) Mounting `Secrets` in the same destination directory where the `Configmap`
files are synced currently generates permission related issues, and passing the
`SubPath` to the `VolumeMount` doesn’t solve the problem. Removing Kolla from
the picture doesn’t add much value rather than keeping it

(\*\*) 04-<service>.conf is generated by the operator, and the `data` field is
nothing more than the concatenation of the data retrieved by the list of the
secrets specified in the service CR


## Resources

The work described in this document is supported by the `Glance` patch that has
been tested via the `meta-operator` driven deployment:

* [Glance PoC](https://github.com/openstack-k8s-operators/glance-operator/pull/221)
