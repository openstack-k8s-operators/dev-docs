# FIPS Support for the openstack operators

## General information

FIPS 140-2 and 140-3 are designed to ensure that cryptographic tools implement
their algorithms properly. In the operator-k8s context, a given operator can be
considered FIPS compliant if the underlying components are FIPS validated.


## Build a FIPS compliant operator

To be FIPS compliant, there are a few requirements and recommendation that
should be take into account:

- Use RHEL based images with openssl to build the binary (alpine or ubuntu images
  don’t have a FIPS validated openssl crypto library)
- Do not use statically linked images
- Set `CGO_ENABLED=1`
- Go containers rely on OpenSSL to detect whether the system is in FIPS mode


### Steps to build a FIPS compatible operator

In the `Dockerfile`, replace both the `GOLANG_BUILDER` and the `OPERATOR_BASE`
image to point to the `go-toolset` and the `ubi-minimal` image:

```bash
ARG GOLANG_BUILDER=registry.access.redhat.com/ubi9/go-toolset:1.19
ARG OPERATOR_BASE_IMAGE=registry.access.redhat.com/ubi9/ubi-minimal:latest
```

In addition, the following parameters are required:

```bash
ARG GO_BUILD_EXTRA_ARGS="-tags strictfipsruntime"
ARG GO_BUILD_EXTRA_ENV_ARGS="CGO_ENABLED=1 GO111MODULE=on"
```

Pass the parameters defined above to the build command:

```bash
RUN if [ -f $CACHITO_ENV_FILE ] ; then source $CACHITO_ENV_FILE ; fi ; env ${GO_BUILD_EXTRA_ENV_ARGS} go build ${GO_BUILD_EXTRA_ARGS} -a -o ${DEST_ROOT}/manager main.go
```


Finally, in the `Makefile`, define build extra variables that can be passed to
the container image build process:

```bash
DOCKER_BUILD_ARGS ?=
..
..
..
.PHONY: docker-build
docker-build: test ## Build docker image with the manager.
	podman build -t ${IMG} . ${DOCKER_BUILD_ARGS}
```

As mentioned earlier, the proposed change is based on:

- `go-toolset`, available as a container image and allows Go to bypass the
   standard library cryptographic routines and call into a `FIPS 140-2`
   validated cryptographic library

- `ubi9/ubi-minimal`, that ships with several FIPS-validated cryptography
   libraries, including OpenSSL


## CI - Check FIPS compliance

The [check-payload tool](https://github.com/openshift/check-payload) can be used
to verify if an operator image is FIPS compliant.
It currently runs as a [stage](https://github.com/openshift/release/pull/47453)
of the existing Prow jobs, where the operator image is built starting from the
current PR.
By default the check doesn't block merging a patch, but it's possible to let the
CI fail if the tool detect a failure when the image is scanned.
To enable the CI failure, edit the `.prow_ci.env` file present in the operator
repository and add:

```bash
export FAIL_FIPS_CHECK=true
```


## Feature reporting

Once an operator and its image are built in a FIPS compliant way, as described
above, and the operator is also honoring the OpenShift Cluster FIPS mode when
deploying its OpenStack services, the operator can announce that is fully FIPS
compliant using an OpenShift specific annotation, as [described in the
OpenShift docs](https://docs.openshift.com/container-platform/4.14/operators/operator_sdk/osdk-generating-csvs.html#osdk-csv-annotations-infra_osdk-generating-csvs).

This means editing the operator's `ClusterServiceVersion` object, for example
in cinder it's in the
`config/manifests/bases/cinder-operator.clusterserviceversion.yaml` file, and
setting the annotation:

```yaml
metadata:
  annotations:
    features.operators.openshift.io/fips-compliant: "true"
```

## Resources

- [check-payload tool](https://github.com/openshift/check-payload)
- [openshift-fips-compliance](https://access.redhat.com/articles/openshift_fips_compliance_faq)
- [Prow job check-payload step](https://github.com/openshift/release/pull/47453)
