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
operator by either setting its controller-manager pod replicas to 0 
or removing the deployment from the CSV entirely so that only your 
local operator will resolve its CR.

## Disabling a service within the CSV

1. Backup the operator's CSV in case you want to restore it later:

```
oc get csv -n openstack-operators <your operator CSV> -o json | \
  jq -r 'del(.metadata.generation, .metadata.resourceVersion)'  > operator_csv.json
```

2. Either patch the CSV for the operator so that it scales down to 0 controller-manager pod replicas:

```
oc patch csv -n openstack-operators <your operator CSV> --type json \
  -p="[{"op": "replace", "path": "/spec/install/spec/deployments/0/spec/replicas", "value": "0"}]"
```

...or remove the deployment outright:

```
oc patch csv -n openstack-operators <your operator CSV> --type json \
  -p="[{"op": "remove", "path": "/spec/install/spec/deployments/0"}]"
```

3. Then [remove the webhooks](https://github.com/openstack-k8s-operators/docs/blob/main/webhooks.md#disabling-webhooks)

4. (Optional) To restore the original OLM-based operator after you are done with local dev/testing:

```
oc patch csv -n openstack-operators <your operator CSV> --type=merge --patch-file=operator_csv.json
```

## An Alternative Approach Provided by ChatGPT

In Kubernetes, CSV (Cluster Service Version) is a Custom Resource Definition (CRD) that enables the operator to manage the lifecycle of a specific application in a Kubernetes cluster. The CSV defines the deployment strategy, dependencies, and upgrade paths for the application.

To remove a deployment from the CSV, you need to update the CSV file and remove the reference to the deployment. Here are the steps to do so:

1. Use the kubectl get csv command to get the name of the CSV that contains the deployment you want to remove.

2. Use the kubectl edit csv &lt;csv-name&gt; command to open the CSV in an editor. This command will open the YAML file for the CSV in the default editor specified by your system.

3. Locate the spec.install.spec.deployments section in the YAML file. This section contains a list of all the deployments managed by the CSV.

4. Remove the deployment that you want to delete from the list. Save the changes and close the editor.

5. Verify that the deployment has been removed by running the kubectl get deployments command. The deployment should no longer be listed.

Note: Removing a deployment from the CSV does not delete the deployment from the cluster. You need to delete the deployment manually using the kubectl delete deployment &lt;deployment-name&gt; command.
