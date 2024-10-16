# Conditions
This doc describes how condiditons should be used in operators managing a podified controlplane (**NOT** osp-director-operator).

## General information
Description of conditions https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties 

**Note**: While it might be useful/easier to manage conditions in a map (easier to access/set) instead of a list, conditions are represented using a list/slice of objects, where each condition has a similar structure. This collection should be treated as a map with a key of type.

## Conditions in openstack-k8s-operators podified controlplane operators
* At the beginning of the reconciliation loop, controller adds a `Ready` condition which reflects the overall state of the service with `Status=Unknown`
* When a controller performs a task, before it starts a step condition gets initialized to `Status=Unknown` e.g. DBReady, DBSync, Endpoint, ServiceRegistered, …
* Conditions should always be re-initialized at the beginning of each reconcile. This is to ensure that changes to the spec
  will reset conditions accordingly.
* At the end of the reconcile loop the overall `Ready` condition gets set to `Status=True/False`, depending if the sub task conditions are fulfilled or not.
* The Ready condition on our CRDs reflect that the expected state described by the Spec field of the CRD is the same as the actual state of the system, i.e. the operator finished reconciling the state of the system according to the Spec.
* If the `Ready` condition is `Status=False` or `Unknown` the service has not been deployed with the currently configured spec.
  - `Status=False` indicates that a condition has not yet been met.
* It is acceptable to set ReadyCondition=True if replicas is 0. Any checks for an active service is actually running should
  check that both replicas is >0 and ReadyCondition=True.

## Controllers and conditions
General behavior of a controller to consume/set conditions:
* The controller should not rely on conditions from previous reconciliations since it is the controller setting the conditions. Instead detect the status of things through introspection at every reconciliation and act accordingly.
* It is acceptable to rely on conditions that the controller set previously as part of the current reconciliation.
* It is acceptable to rely on conditions of other API objects during a reconciliation to expect a service state, e.g. if keystoneapi `Ready` condition is `Status=True`, other services can expect that keystone is up and running.

## Condition type for openstack-k8s-operators API objects
| Parameter | Description |
| --- | --- |
| `Type` | Type of condition in CamelCase |
| `Status` | Status of the condition, one of `True`, `False`, `Unknown`. |
| `Severity` | Severity provides a classification of Reason code, so the current situation is immediately understandable and could act accordingly. It is meant for situations where `Status=False` and it should be indicated if the issue is just informational, warning (next reconciliation might fix it) or an error (e.g. DB create issue and no actions to automatically resolve the issue can/should be done). |
| `LastTransitionTime` | Last time the condition transitioned from one status to another. This should be when the underlying condition changed. If that is not known, then using the time when the API field changed is acceptable. |
| `Reason` | The reason for the condition's last transition in CamelCase. |
| `Message` | A human readable generic message indicating details about the transition. |

### Type
This does not reflect a complete list of condition types, it is just to illustrate how to use conditions.

| Condition Type | Description |
| --- | --- |
| `ReadyCondition` | All APIs have at least one `ReadyCondition` condition type that summarizes the overall operational state of the API object. Each CRD has a `IsReady()` method which returns a bool of the status of the instance/Ready condition https://github.com/openstack-k8s-operators/keystone-operator/blob/82267e337f6a24ac86987be0c13275b22a838d6b/api/v1beta1/keystoneapi\_types.go#L221 |
| `InputReadyCondition` | Condition which indicates if all required input sources are available, like e.g. secret holding passwords, other config maps providing input for the service. |
| `ServiceConfigReadyCondition` | Condition which indicates that all service config got rendered ok from the templates and stored in the ConfigMap |
| `DBReadyCondition` | This condition is mirrored from the `Ready` condition in the `mariadbdatabase` ref object to the service API. |
| `DBSyncReadyCondition` | `Status=True` condition when dbsync job completed ok |
| `ExposeServiceReadyCondition` | `Status=True` condition when service/routes to expose the service created ok |
| `BootstrapReadyCondition` | `Status=True` condition when bootstrap job completed ok |
| `KeystoneServiceReadyCondition` | This condition is mirrored from the `Ready` condition in the keystoneservice ref object to the service API. |

### Severity
| Severity | Description |
| --- | --- |
| `Error` | condition with `Status=False` is an error |
| `Warning` | condition with `Status=False` is a warning |
| `Info` | condition with `Status=False` is informative |
| “” (none) | should apply only to conditions with `Status=True` |

### Generic Reasons

#### DBcreate
| Condition Reason | Description |
| --- | --- |
| `RequestedReason` | (`Severity=Info`) documents a condition not in `Status=True` because the DB is currently being created. |
| `CreationFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an issue happened during DB creation.|
| `ReadyReason` | documents a condition in `Status=True` when the DB got created and the DB is ready to be used. |

#### DBsync
| Condition Reason | Description |
| --- | --- |
| `CreatedReason` | (`Severity=Info`) documents a condition not in `Status=True` because the DBsync job got created, but is still running. |
| `CreationFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an issue happened during DBsync job. |
| `ReadyReason` | documents a condition in `Status=True` when the DBsync job finished ok, or does not need to run as it ran before. |

#### Expose service
| Condition Reason | Description |
| --- | --- |
| `CreationFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an issue happened during creating routes/services to expose the service. |
| `ReadyReason` | documents a condition in `Status=True` when the routes/services got created. |

#### Bootstrap
| Condition Reason | Description |
| --- | --- |
| `CreatedReason` | (`Severity=Info`) documents a condition not in Status=True because the bootstrap job got created, but is still running. |
| `CreationFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an issue happened during bootstrap job. |
| `ReadyReason` | documents a condition in `Status=True` when the Bootstrap job finished ok, or does not need to run as it ran before. |

#### Deployment
|Condition Reason | Description |
| --- | --- |
| `CreatedReason` | (`Severity=Info`) documents a condition not in `Status=True` because the service deployment got created, but has not yet reached the min number of one ReadyCount. |
| `CreationFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an issue happened during deployment creation.|
| `ReadyReason` | documents a condition in `Status=True` when the deployment got created **AND** there is at least one pod up to answer service requests (`ReadyCount` >= 1). |

### Service specific Reasons

#### KeystoneAPI Reasons
| Condition Reason | Description |
| --- | --- |
| `FernetCreatedReason` | (`Severity=Info`) documents a condition not in `Status=True` because that the fernet tokens got created. |

#### KeystoneService Reasons
| Condition Reason | Description |
| --- | --- |
| `ServiceCreatedReason` | (`Severity=Info`) documents a condition not in `Status=True` because the service got created in keystone. |
| `ServiceFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an error happened during service registration. |
| `EndpointCreatedReason` | (`Severity=Info`) documents a condition not in `Status=True` because the service endpoints got registered in keystone. |
| `EndpointFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an error happened during service endpoint registration. |
| `RoleCreatedReason` | (`Severity=Info`) documents a condition not in `Status=True` because the role was created ok in keystone. |
| `RoleFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an error happened during creating the role. |
| `UserCreatedReason` | (`Severity=Info`) documents a condition not in `Status=True` because the user was created ok in keystone. |
| `UserFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an error happened during creating the user. |
| `AssignUserRoleCreatedReason` | (`Severity=Info`) documents a condition not in `Status=True` because the user got assigned to the role ok in keystone. |
| `AssignUserRoleFailedReason` | (`Severity=Error`) documents a condition not in `Status=True` because an error happened during assigning a role to the user. |
