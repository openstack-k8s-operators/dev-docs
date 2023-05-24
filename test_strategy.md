# Test strategy

## Design test
These are the checks and tests encouraged to be provided and maintained by the
developers of a given operator. This is intentionally focusing on the
bottom of the
[test pyramid](https://martinfowler.com/articles/practical-test-pyramid.html).

1. A common set of static analysis tools, stylers, linters aggregated into a
single [pre-commit](https://pre-commit.com/) run both locally and in CI:
   * go standard tools: fmt, vet, mod tidy
   * [golangci-lint](https://golangci-lint.run/usage/linters/): a meta linter
   with a tons of linters included but only a small subset enabled by default
   * [operator-lint](https://github.com/gibizer/operator-lint): a home grew
   linter automating operator development and testing specific rules

   Each operator can extend the list of linters but we need a common minimum
   set across each operators.

2. Unit tests covering functions or types using go’s standard
   [testing](https://go.dev/doc/tutorial/add-a-test) lib and
   [gomega](https://onsi.github.io/gomega/)

3. Component tests (also called functional or service tests) covering the
   reconciliation of a single CRD or a set of CRDs from the same operator
   using [EnvTest](./envtest.md)

4. Integration test using [kuttl](./kuttl_tests.md) on OCP
   * Covering a single operator and its dependencies (e.g. placement-operator
   tests will depend on keystone-operator and mariadb-operator)
   * Covering our integrated set of operators via openstack-operator

## Repository specific rules

### nova-operator
* Each PR changing implementation logic have to include relevant EnvTest
coverage
