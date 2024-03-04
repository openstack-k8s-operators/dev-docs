# Troubleshooting guide

## Crashing Pods

Certain problems in the deployment can cause Pods to crash and terminate
repeatedly and ending up in `CrashLoopBackOff` state. In this state it is not
possible to directly log into the Pod via `oc rsh` to troubleshoot the issue.
However you can use `oc debug` to duplicate the crashing Pod without the
container command, probes, and labels and get a shell into it. If you need to
route traffic to the Pod then you can use `--keep-labels=true` in the
`oc debug` command to keep the duplicated Pod as a backed of the related
Service.

The debug command can be used not only on a specific `Pod` (`oc debug
cinder-api-0`) but also on any resource that creates pods such as
`Deployments`, `StatefulSets`, etc. as well as `Nodes`.  For example: `oc debug
StatefulSet/cinder-scheduler`.

By default `oc debug` will run `/bin/sh` on the first container of the pod, but
we can easily specify a different container to debug.  This is useful for pods
like the API where the first container is tailing a file and its the second
container the one we usually want to debug.  For example: `oc debug -c
cinder-api pod/cinder-api-0`.

We can use these last 2 functionalities, debugging non `Pod` resources and
specifying the container, to debug Init Containers.  For example: `oc debug
-c init Deployment/dnsmasq-dns`.

Please check the `oc debug --help` output for other available functionality
such as `--as-root`, `--as-user`, `--to-namespace`, `--node-name`, etc.
