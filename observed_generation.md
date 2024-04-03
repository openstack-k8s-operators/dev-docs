# Generation and ObservedGeneration

## Problem
A client should be able to decide if the reconciliation of the latest `Spec`
update has been finished by looking at the `Status` field of the CR. The
`Status.Conditions`, and especially the `Ready` condition in that list, can be
used to determine the result of the last reconciliation run. However the
conditions alone cannot tell which version of the `Spec` was used for the last
reconciliation, they cannot indicate if there is a newer `Spec` update that has
not seen by the controller and therefore hasn't reconciled yet.


## Solution
K8s maintains a `Generation` field for each CR. When the `Spec` field of the
CR is updated the `Generation` of the CR is automatically incremented.

To allow the client to decide if the `Status` field talks about the latest
version of the `Spec` the `Status` field should tell which version, generation,
the `Status` is referring to. This is implemented by the
`Status.ObservedGeneration` field. This field contains the `Generation` value
of the `Spec` the controller last saw and therefore the `Spec` version the
`Status` is referring to.

## Usage
If a client updates the `Spec` field of a CR and wants to decide when such
`Spec` changes are applied to the running system, then the client needs to look
at two fields in the CR `Status`:
* If the `Status.ObservedGeneration == Generation` then the client can be sure
that the controller seen the last `Spec` version
* If the `Ready` condition's State is `True` in the `Status.Conditions` list
then the  client can be sure that the last reconciliation finished
successfully.

```golang
if subCR.Generation == subCR.Status.ObservedGeneration &&
	subCR.Status.Conditions.IsTrue(condition.ReadyCondition) {
        // subCR is successfully reconciled the latest Spec update
}
```

If the `Generation > Status.ObservedGeneration` then the subCR controller
haven't seen the new subCR.Spec so the client needs to wait.

However `Generation < Status.ObservedGeneration` case is also a possibility
even if it is a strange one. At a first look one can say that this means
the `Spec`'s generation went backwards and that should not happen. However it
can happen in a valid case too. Consider the following sequence that leads to
this situation:
1. our client updates the subCR and reads back that its `Generation` is now N
2. another client updates the subCR so its `Generation` is now N+1
3. the subCR controller reconciles the latest spec and sets
`ObservedGeneration` to N+1
4. our client reads the `subCR.Status` and sees `ObservedGeneration` N+1 while
it sees `Generation` as N. Note that `Status` is a subresource in the API so it
can be read independently.
In this case our client cannot decide if the `subCR.Spec` is according to its
needs, maybe the other client set something on the `Spec` that our controller
needs to reset. So in this case our client should not state that the subCR is
up to date but instead re-read the spec (if the client is in a reconciler the
requeue the reconcile) and update the subCR.Spec again if needed.

## Implementation
The `Generation` field is implemented automatically.

The `Status.ObservedGeneration` needs to be implemented by every controller
individually. This field needs to be bumped to the value of the `Generation`
field at the start of each `Reconcile` run, after the `Status.Conditions`
field is reset to signal that a new `Spec` version is being reconciled and
therefore the old condition values are not valid any more.

## When a CR depends on one or more sub CRs
We have multiple cases when the Readiness of a parent CR depends on the status
of other sub CRs (e.g. `Glance` CR `Status` depends on one or more GlanceAPI
CRs `Status`). If the above implementation is in place then the
`ObservedGeneration` of the parent CR will not depend on the
`ObservedGeneration` of the sub CRs. But the Readiness (the Ready=true state)
of the parent CR depends on the `subCR.Status` fields (mostly the
`Status.Conditions`) and therefore it needs to consider what `Spec`
`Generation` those `Status` fields are referring to. So the
`subCR.Status.ObservedGeneration` needs to be included in the parent CR's
Readiness calculation. This is needed as the subCR's `Ready` condition alone
cannot tell if the controller of the subCR see the latest `Spec` update from
the controller of the parent CR.

## When a CR depends on Pods
We have CRs that are managing `Pods` via `Deployment`, `StatefulSet` or other
similar deployment resources. These deployment resources have
`ObservedGeneration` as well. The `ObservedGeneration` of the CR should not
depend on the `ObservedGeneration` of the deployment resource. But the
Readiness of the CR needs to consider `Status` fields of the deployment
resource like `ReadyCount`. However that `ReadyCount` might refer to
an older `Spec` generation therefore the CR needs to also check the
`ObservedGeneration` of the deployment resource when using the `ReadyCount`
(or other `Status` fields) to calculate it's Readiness.

