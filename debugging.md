# Debugging Go based operator

## Debugging operator in vscode

### Prerequisite

* install [current used go version](https://github.com/openstack-k8s-operators/install_yamls/blob/main/devsetup/vars/default.yaml)
* install [current used operator-sdk version](https://github.com/openstack-k8s-operators/install_yamls/blob/main/devsetup/vars/default.yaml)
* install vscode
* make sure 
  [go extension for vscode](https://marketplace.visualstudio.com/items?itemName=golang.Go)
  is installed
* install [delve debugger for go](https://github.com/go-delve/delve)

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

* Deploy the needed operator CRDs/manifests either manually or install and
  uninstall via olm (optional, see `tasks.json` below)

```bash
oc login -u kubeadmin -p '12345678' https://api.ostest.test.metalkube.org:6443
make install
```

### Run debugger via vscode

When imported the base dir in the workspace.

**Create `tasks.json` to automate CRD/manifests install**
A `.vscode/tasks.json` needs to be created which then can be referenced from the
debug configuration. Create the `.vscode/tasks.json` with `Terminal`
-> `Configure Tasks...` -> `Go: build package`

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Run make",
      "type": "shell",
      "command": "make install manifests generate fmt vet",
      "group": "build",
      "presentation": {
        "reveal": "always",
        "panel": "new"
      },
      "options": {
        "cwd": "${fileWorkspaceFolder}/",
        "env": {
          "OPERATOR_TEMPLATES": "./templates",
          "PATH": "~/.crc/bin/oc:${env:PATH}"
        }
      },
      "problemMatcher": []
    }
  ]
}
```

**Create `launch.json` for the debugger**
A `.vscode/launch.json` needs to be created which instructs vscode how to start
the debugger. Create the `.vscode/launch.json` with `Run` ->
`Add configuration` -> `Go: Launch Package`.

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Package",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${fileWorkspaceFolder}/main.go",
      "args": [
        "-metrics-bind-address",
        ":${env:METRICS_PORT}",
        "-health-probe-bind-address",
        ":${env:HEALTH_PORT}"
      ],
      "cwd": "${fileWorkspaceFolder}",
      "preLaunchTask": "Run make",
      "env": {
        "WATCH_NAMESPACE": "openstack",
        "OPERATOR_TEMPLATES": "./templates",
        "ENABLE_WEBHOOKS": "false",
        "METRICS_PORT": "8082",
        "HEALTH_PORT": "8083"
      }
    }
  ]
}
```

* program points to the main package dir or `main.go` file.
* Set any required env vars

Before running the operator you will need to configure the system to allow
the communication between the local running operator and the services running
on the remote cluster as documented
[here](./running_local_operator.md#allow-local-running-operator-to-connect-to-k8s-services)

Make sure you logged into the OCP cluster, set break points where required and
start the debugger with either `F5` or `Run` -> `Start Debugging`.

Check the Debug Console(`Ctrl+Shift+Y`) that the operator starts ok.

Happy debugging!
