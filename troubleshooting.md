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