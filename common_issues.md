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
