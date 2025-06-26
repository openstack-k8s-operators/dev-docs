# Using pprof to connect to openstack-k8s-operators controller managers

## step 1 edit CSV/disable initialization controller-operator
 edit CSV and disable the initialization resource manager for openstack-controller-operator
 by setting replicas to 0

## step 2 enable pprof bind address
 add --pprof-bind-address to the deployment for the manager you want to inspect. This could be any
 operator. Example: infra-operator, openstack-operator, etc.

## step 3 port forward

```bash
oc port-forward infra-operator-controller-manager-58b5b5cb84-mpwq2 -n openstack-operators 8082
```

## step 4 connect pprof
```bash
go tool pprof http://localhost:8082/debug/pprof/heap
```

## step 5 (run misc pprof commands)
useful commands:
 * top
 * list
 * web

useful modes (switch between these and run the above commands):
 * inuse\_space: The amount of memory currently in use (allocated but not yet freed). This is often the most useful for identifying memory leaks.
 * inuse\_objects: The number of objects currently in use.
 * alloc\_space: The total amount of memory allocated over the lifetime of the application (including memory that has been freed).
 * alloc\_objects: The total number of objects allocated over the lifetime of the application.

Example leak detection:

```bash
curl -s http://localhost:8082/debug/pprof/heap > heap1.prof
curl -s http://localhost:8082/debug/pprof/heap > heap2.prof
go tool pprof -http=:8081 --base heap1.prof heap2.prof
```
