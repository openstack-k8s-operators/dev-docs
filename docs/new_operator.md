# Start a new service operator
To start a new service operator from scratch and allign to the work done in the keystone-operator do:

## Start new project
```bash
mkdir placement-operator && cd placement-operator
operator-sdk init --domain openstack.org --project-name placement-operator --repo github.com/openstack-k8s-operators/placement-operator
```

## Init the git repo
```bash
git init
git add * 
git add .gitignore
git add .dockerignore
git commit -m "init"
```

## Update .gitignore file
```
diff --git a/.gitignore b/.gitignore
index c0a7a54..437eeac 100644
--- a/.gitignore
+++ b/.gitignore
@@ -18,8 +18,16 @@ testbin/*
 
 !vendor/**/zz_generated.*
 
+#Operator SDK generated files
+/bundle/
+bundle.Dockerfile
+config/manager/kustomization.yaml
+
 # editor and IDE paraphernalia
 .idea
 *.swp
 *.swo
 *~
+
+# Common CI tools repository
+CI_TOOLS_REPO
```

```bash
$ git add .gitignore && git commit -m "update .gitignore"
```

## Update Makefile

```
diff --git a/Makefile b/Makefile
index 49afd68..091de81 100644
--- a/Makefile
+++ b/Makefile
@@ -29,7 +29,7 @@ BUNDLE_METADATA_OPTS ?= $(BUNDLE_CHANNELS) $(BUNDLE_DEFAULT_CHANNEL)
 #
 # For example, running 'make bundle-build bundle-push catalog-build catalog-push' will build and push both
 # openstack.org/placement-operator-bundle:$VERSION and openstack.org/placement-operator-catalog:$VERSION.
-IMAGE_TAG_BASE ?= openstack.org/placement-operator
+IMAGE_TAG_BASE ?= quay.io/$(USER)/placement-operator
 
 # BUNDLE_IMG defines the image:tag used for the bundle.
 # You can use it as an arg. (E.g make bundle-build BUNDLE_IMG=<some-registry>/<project-name-bundle>:<tag>)
@@ -118,11 +118,11 @@ run: manifests generate fmt vet ## Run a controller from your host.
 
 .PHONY: docker-build
 docker-build: test ## Build docker image with the manager.
-       docker build -t ${IMG} .
+       podman build -t ${IMG} .
 
 .PHONY: docker-push
 docker-push: ## Push docker image with the manager.
-       docker push ${IMG}
+       podman push ${IMG}
 
 ##@ Deployment
 
@@ -185,7 +185,7 @@ bundle: manifests kustomize ## Generate bundle manifests and metadata, then vali
 
 .PHONY: bundle-build
 bundle-build: ## Build the bundle image.
-       docker build -f bundle.Dockerfile -t $(BUNDLE_IMG) .
+       podman build -f bundle.Dockerfile -t $(BUNDLE_IMG) .
 
 .PHONY: bundle-push
 bundle-push: ## Push the bundle image.
@@ -213,7 +213,7 @@ endif
 BUNDLE_IMGS ?= $(BUNDLE_IMG)
 
 # The image tag given to the resulting catalog image (e.g. make catalog-build CATALOG_IMG=example.com/operator-catalog:v0.2.0).
-CATALOG_IMG ?= $(IMAGE_TAG_BASE)-catalog:v$(VERSION)
+CATALOG_IMG ?= $(IMAGE_TAG_BASE)-index:v$(VERSION)
 
 # Set CATALOG_BASE_IMG to an existing catalog image tag to add $BUNDLE_IMGS to that image.
 ifneq ($(origin CATALOG_BASE_IMG), undefined)
@@ -225,9 +225,44 @@ endif
 # https://github.com/operator-framework/community-operators/blob/7f1438c/docs/packaging-operator.md#updating-your-existing-operator
 .PHONY: catalog-build
 catalog-build: opm ## Build a catalog image.
-       $(OPM) index add --container-tool docker --mode semver --tag $(CATALOG_IMG) --bundles $(BUNDLE_IMGS) $(FROM_INDEX_OPT)
+       $(OPM) index add --container-tool podman --mode semver --tag $(CATALOG_IMG) --bundles $(BUNDLE_IMGS) $(FROM_INDEX_OPT)
 
 # Push the catalog image.
 .PHONY: catalog-push
 catalog-push: ## Push a catalog image.
        $(MAKE) docker-push IMG=$(CATALOG_IMG)
+
+
+# CI tools repo for running tests
+CI_TOOLS_REPO := https://github.com/openstack-k8s-operators/openstack-k8s-operators-ci
+CI_TOOLS_REPO_DIR = $(shell pwd)/CI_TOOLS_REPO
+.PHONY: get-ci-tools
+get-ci-tools:
+       if [ -d  "$(CI_TOOLS_REPO_DIR)" ]; then \
+               echo "Ci tools exists"; \
+               pushd "$(CI_TOOLS_REPO_DIR)"; \
+               git pull --rebase; \
+               popd; \
+       else \
+               git clone $(CI_TOOLS_REPO) "$(CI_TOOLS_REPO_DIR)"; \
+       fi
+
+# Run go fmt against code
+gofmt: get-ci-tools
+       $(CI_TOOLS_REPO_DIR)/test-runner/gofmt.sh
+
+# Run go vet against code
+govet: get-ci-tools
+       $(CI_TOOLS_REPO_DIR)/test-runner/govet.sh
+
+# Run go test against code
+gotest: get-ci-tools
+       $(CI_TOOLS_REPO_DIR)/test-runner/gotest.sh
+
+# Run golangci-lint test against code
+golangci: get-ci-tools
+       $(CI_TOOLS_REPO_DIR)/test-runner/golangci.sh
+
+# Run go lint against code
+golint: get-ci-tools
+       PATH=$(GOBIN):$(PATH); $(CI_TOOLS_REPO_DIR)/test-runner/golint.sh
```

```bash
git add Makefile && git commit -m "update Makefile"
```

## Update Dockerfile
```
diff --git a/Dockerfile b/Dockerfile
index 456533d..6afe478 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -1,27 +1,75 @@
+ARG GOLANG_BUILDER=golang:1.17
+ARG OPERATOR_BASE_IMAGE=gcr.io/distroless/static:nonroot
+
 # Build the manager binary
-FROM golang:1.17 as builder
+FROM $GOLANG_BUILDER AS builder
+
+#Arguments required by OSBS build system
+ARG CACHITO_ENV_FILE=/remote-source/cachito.env
+
+ARG REMOTE_SOURCE=.
+ARG REMOTE_SOURCE_DIR=/remote-source
+ARG REMOTE_SOURCE_SUBDIR=
+ARG DEST_ROOT=/dest-root
+
+ARG GO_BUILD_EXTRA_ARGS=
+
+COPY $REMOTE_SOURCE $REMOTE_SOURCE_DIR
+WORKDIR $REMOTE_SOURCE_DIR/$REMOTE_SOURCE_SUBDIR
+
+RUN mkdir -p ${DEST_ROOT}/usr/local/bin/
 
-WORKDIR /workspace
-# Copy the Go Modules manifests
-COPY go.mod go.mod
-COPY go.sum go.sum
 # cache deps before building and copying source so that we don't need to re-download as much
 # and so that source changes don't invalidate our downloaded layer
-RUN go mod download
+RUN if [ ! -f $CACHITO_ENV_FILE ]; then go mod download ; fi
 
-# Copy the go source
-COPY main.go main.go
-COPY api/ api/
-COPY controllers/ controllers/
+# Build manager
+RUN if [ -f $CACHITO_ENV_FILE ] ; then source $CACHITO_ENV_FILE ; fi ; CGO_ENABLED=0  GO111MODULE=on go build ${GO_BUILD_EXTRA_ARGS} -a -o ${DEST_ROOT}/manager main.go
 
-# Build
-RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -o manager main.go
+RUN cp -r templates ${DEST_ROOT}/templates
 
 # Use distroless as minimal base image to package the manager binary
 # Refer to https://github.com/GoogleContainerTools/distroless for more details
-FROM gcr.io/distroless/static:nonroot
+FROM $OPERATOR_BASE_IMAGE
+
+ARG DEST_ROOT=/dest-root
+# NONROOT default id https://github.com/GoogleContainerTools/distroless/blob/main/base/base.bzl#L8=
+ARG USER_ID=65532
+
+ARG IMAGE_COMPONENT="placement-operator-container"
+ARG IMAGE_NAME="placement-operator"
+ARG IMAGE_VERSION="1.0.0"
+ARG IMAGE_SUMMARY="Placement Operator"
+ARG IMAGE_DESC="This image includes the placement-operator"
+ARG IMAGE_TAGS="cn-openstack openstack"
+
+### DO NOT EDIT LINES BELOW
+# Auto generated using CI tools from
+# https://github.com/openstack-k8s-operators/openstack-k8s-operators-ci
+
+# Labels required by upstream and osbs build system
+LABEL com.redhat.component="${IMAGE_COMPONENT}" \
+      name="${IMAGE_NAME}" \
+      version="${IMAGE_VERSION}" \
+      summary="${IMAGE_SUMMARY}" \
+      io.k8s.name="${IMAGE_NAME}" \
+      io.k8s.description="${IMAGE_DESC}" \
+      io.openshift.tags="${IMAGE_TAGS}"
+### DO NOT EDIT LINES ABOVE
+
+ENV USER_UID=$USER_ID \
+    OPERATOR_TEMPLATES=/usr/share/placement-operator/templates/
+
 WORKDIR /
-COPY --from=builder /workspace/manager .
-USER 65532:65532
+
+# Install operator binary to WORKDIR
+COPY --from=builder ${DEST_ROOT}/manager .
+
+# Install templates
+COPY --from=builder ${DEST_ROOT}/templates ${OPERATOR_TEMPLATES}
+
+USER $USER_ID
+
+ENV PATH="/:${PATH}"
 
 ENTRYPOINT ["/manager"]
```

```bash
git add Dockerfile && git commit -m "update Dockerfile"
```

## Github workflows
Add .github folder and workflows from e.g. [keystone-operator](https://github.com/openstack-k8s-operators/keystone-operator/tree/master/.github) and update to the new operator.

## Create ClusterServiceVersion for the operator
In `config/manifests/bases/` create `placement-operator.clusterserviceversion.yaml`, like [keystone-operator.clusterserviceversion.yaml](https://github.com/openstack-k8s-operators/keystone-operator/blob/master/config/manifests/bases/keystone-operator.clusterserviceversion.yaml)

## Create templates folder holding scripts/config templates
```bash
mkdir templates
```

## Create the first API & controller
```bash
operator-sdk create api --group placement --version v1beta1 --kind PlacementAPI --resource --controller
```

## Create pkg folder holding operator specific packages
```bash
mkdir -p pkg/placement
```

## Update API and controller as needed
- api/v1beta1/placementapi_types.go
- controllers/placementapi_controller.go

See https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/ for a tutorial, or take the keystone-operator as a reference.

## Make `api` a separate go module
- Add a `go.mod` file to the `./api` directory to make it a separate go module.
- Add a
  ```
  replace github.com/openstack-k8s-operators/<your operator>/api => ./api
  ```
  line to the main `go.mod` file.

This way other operators creating CRs of this operator only need to depend
on a small subset of the operator implementation.

## Test the operator
### Run the operator local without need to buld image
A pre req is that the CRDs, RBAC resources got all created in the cluster.

```bash
OPERATOR_TEMPLATES=./templates make run
```

### Create operator image and push to custom registry
```bash
IMAGE_TAG_BASE=quay.io/<user>/placement-operator VERSION=0.0.1 IMG=$IMAGE_TAG_BASE:v$VERSION make manifests build docker-build docker-push bundle bundle-build bundle-push catalog-build catalog-push
```
