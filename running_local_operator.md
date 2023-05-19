# Running a local copy of an operator for development without conflicts

You can run a local copy of an operator for development using commands
like the following in that operator's directory.
```
make manifests generate build
OPERATOR_TEMPLATES=$PWD/templates ./bin/manager
```
However, the operator deployed by the meta operator (from when you
ran `make openstack`) will conflict with your local copy. E.g. both
operators might try to resolve the same CR.

To avoid the conflict, scale down the operator deployed by the meta
operator by removing it from the CSV so that only your local operator
will resolve its CR.

## Removing a service from the CSV

Define an operator name and query the CSV to identify the index of the service you want to scale down  

```
export OPERATOR_NAME="openstack-ansiblee-operator"
OPERATOR_INDEX=(oc get csv openstack-operator.v0.0.1 -o json | jq -r '.spec.install.spec.deployments | map(.name == $ENV.OPERATOR_NAME + "-controller-manager") | index(true)'
```
Update the CSV
```
oc patch csv openstack-baremetal-operator.v0.0.1 --type json \
  -p="[{"op": "remove", "path": "/spec/install/spec/deployments/${OPERATOR_INDEX}"}]"
```

Unset variables
```
unset OPERATOR_NAME OPERATOR_INDEX
```

## Explanation Provided by ChatGPT

In Kubernetes, CSV (Cluster Service Version) is a Custom Resource Definition (CRD) that enables the operator to manage the lifecycle of a specific application in a Kubernetes cluster. The CSV defines the deployment strategy, dependencies, and upgrade paths for the application.

To remove a deployment from the CSV, you need to update the CSV file and remove the reference to the deployment. Here are the steps to do so:

1. Use the kubectl get csv command to get the name of the CSV that contains the deployment you want to remove.

2. Use the kubectl edit csv &lt;csv-name&gt; command to open the CSV in an editor. This command will open the YAML file for the CSV in the default editor specified by your system.

3. Locate the spec.install.spec.deployments section in the YAML file. This section contains a list of all the deployments managed by the CSV.

4. Remove the deployment that you want to delete from the list. Save the changes and close the editor.

5. Verify that the deployment has been removed by running the kubectl get deployments command. The deployment should no longer be listed.

Note: Removing a deployment from the CSV does not delete the deployment from the cluster. You need to delete the deployment manually using the kubectl delete deployment &lt;deployment-name&gt; command.
