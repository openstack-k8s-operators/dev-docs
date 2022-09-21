
# Implementation guidelines
This is a collection of guidelines for developing the operators. The goal
here is to keep the codebase consistent and help spread good patterns
between the developers.

You can also consult with the [OpenShift Operator Best Practices](https://redhat-openshift-ecosystem.github.io/community-operators-prod/best-practices/).

## Logging
When you log from any of the controller implementations use the lib-common
`utils.LogForObject` and `utils.LogErrorForObject` calls. These will make sure
that the log line has enough information to identify the CR being reconciled.
These calls need a lib-common `Helper` object. So in the short time window
before such an object is created at the start of the reconcile call logging
should be done via the logger provided by the `log.FromContext(ctx)` call.

## Clients
The reconcile loop has access to two k8s clients `client` and `kclient`. As a
rule of thumb use the `client` to query, create, and update k8s objects in the
cluster. The `kclient` is used by lib-common in some edge cases but no new
usage of the `kclient` should be introduced if possible.
