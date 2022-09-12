Common issues
=============

* **Problem:** Operator compilation fails with `reading
  github.com/openshift/api/go.mod at revision v3.9.0: unknown revision
  v3.9.0`.

  **Solution:** Use `go env GOPROXY` to verify that your proxy setting
  is `https://proxy.golang.org,direct`. If not, set it:

  ```
  go env -w GOPROXY="https://proxy.golang.org,direct"
  ```

* **Problem:** Operator compilation fails with `go: github.com/openstack-k8s-operators/lib-common/modules/common@v0.0.0-00010101000000-000000000000: invalid version: unknown revision 000000000000`.

  **Solution:** Add a line like the following to go.mod. Be sure to
  replace `20220909175216-e774739df18a` with the latest version of
  lib-common. The example below was written when
  `20220909175216-e774739df18a` was latest so it will change.
  ```
  replace github.com/openstack-k8s-operators/lib-common/modules/common v0.0.0-00010101000000-000000000000 => github.com/openstack-k8s-operators/lib-common/modules/common v0.0.0-20220909175216-e774739df18a
  ```
  After adding the above, `make` should suceed in the go
  workspace. Remove this line prior to submitting a PR as we do not
  want to hard code an old version which will not be automatically
  updated.
