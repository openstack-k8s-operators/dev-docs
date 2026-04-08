# Building Operator Updates for OLM

This guide explains how to build operator versions that support OLM (Operator Lifecycle Manager) upgrades.

## Prerequisites

- An existing operator version is deployed via OLM
- The previous version's bundle exists in a container registry
- You have push access to your container registry

## Simple Update (Single Version Upgrade)

When creating your first update from an upstream version to your custom version:

**Example: Upgrading from upstream v0.6.0 to custom v0.6.1**

```bash
IMAGENAMESPACE=userx \
IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.0 \
VERSION=0.6.1 \
IMG=quay.io/userx/openstack-operator:v0.6.1 \
make manifests generate bindata build docker-build docker-push bundle bundle-build bundle-push catalog-build catalog-push
```

**What this does:**
- Creates v0.6.1 bundle with `replaces: openstack-operator.v0.6.0` in the CSV
- `PREV_BUNDLE_IMG` defaults to `quay.io/openstack-k8s-operators/openstack-operator-bundle:latest` (upstream)
- Builds catalog containing both v0.6.0 (upstream) and v0.6.1 (yours)
- OLM will detect the upgrade path: v0.6.0 → v0.6.1

## Multi-Version Updates (Incremental Upgrades)

When creating subsequent versions on top of your own versions:

**Example: Upgrading from custom v0.6.1 to custom v0.6.2**

```bash
IMAGENAMESPACE=userx \
IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.1 \
VERSION=0.6.2 \
IMG=quay.io/userx/openstack-operator:v0.6.2 \
PREV_BUNDLE_IMG=quay.io/userx/openstack-operator-bundle:v0.6.1 \
CATALOG_BASE_IMG=quay.io/userx/openstack-operator-index:v0.6.1 \
make manifests generate bindata build docker-build docker-push bundle bundle-build bundle-push catalog-build catalog-push
```

**What this does:**
- Creates v0.6.2 bundle with `replaces: openstack-operator.v0.6.1` in the CSV
- `PREV_BUNDLE_IMG` points to your v0.6.1 bundle (not upstream)
- `CATALOG_BASE_IMG` uses your existing v0.6.1 catalog as a base
- Builds catalog containing v0.6.0 → v0.6.1 → v0.6.2
- OLM will detect the upgrade path from v0.6.1 to v0.6.2

## Key Variables Explained

| Variable | Purpose | Example |
|----------|---------|---------|
| `REPLACES` | Previous CSV name that this version replaces | `openstack-operator.v0.6.1` |
| `VERSION` | New version being built | `0.6.2` |
| `IMG` | New operator image | `quay.io/userx/openstack-operator:v0.6.2` |
| `PREV_BUNDLE_IMG` | Bundle image of the version being replaced (used with `REPLACES`) | `quay.io/userx/openstack-operator-bundle:v0.6.1` |
| `CATALOG_BASE_IMG` | Existing catalog to build upon (for incremental builds) | `quay.io/userx/openstack-operator-index:v0.6.1` |
| `CATALOG_BUNDLE_IMGS` | Comma-separated list of all bundles to include (for building complete upgrade path from scratch) | `"bundle1:latest,bundle2:v0.6.1,bundle3:v0.6.2"` |

## Variable Defaults

If not specified:
- `PREV_BUNDLE_IMG` defaults to `quay.io/openstack-k8s-operators/openstack-operator-bundle:latest` (upstream)
- `CATALOG_BASE_IMG` - if not set, builds a fresh catalog from scratch
- `CATALOG_BUNDLE_IMGS` - auto-computed based on other variables (see strategies below)

## Catalog Building Strategies

There are three approaches to building catalogs:

### 1. Incremental Build (Recommended for iterative development)

Use `CATALOG_BASE_IMG` to add the new bundle to an existing catalog:

```bash
IMAGENAMESPACE=userx \
IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.2 \
VERSION=0.6.3 \
IMG=quay.io/userx/openstack-operator:v0.6.3 \
PREV_BUNDLE_IMG=quay.io/userx/openstack-operator-bundle:v0.6.2 \
CATALOG_BASE_IMG=quay.io/userx/openstack-operator-index:v0.6.2 \
make catalog-build catalog-push
```

**Pros:** Fast, builds on existing catalog
**Cons:** Requires previous catalog to be available

### 2. Auto From Scratch (Single Previous Version)

Use `REPLACES` without `CATALOG_BASE_IMG` to include one previous version:

```bash
IMAGENAMESPACE=userx \
IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.2 \
VERSION=0.6.3 \
IMG=quay.io/userx/openstack-operator:v0.6.3 \
PREV_BUNDLE_IMG=quay.io/userx/openstack-operator-bundle:v0.6.2 \
make catalog-build catalog-push
```

**Pros:** Builds from scratch, no base image needed
**Cons:** Only includes one upgrade step (v0.6.2 → v0.6.3), not the full upgrade path

### 3. Full From Scratch (Complete Upgrade Path)

Specify all bundles explicitly with `CATALOG_BUNDLE_IMGS`:

```bash
IMAGENAMESPACE=userx \
IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
VERSION=0.6.3 \
IMG=quay.io/userx/openstack-operator:v0.6.3 \
CATALOG_BUNDLE_IMGS="quay.io/openstack-k8s-operators/openstack-operator-bundle:latest,quay.io/userx/openstack-operator-bundle:v0.6.1,quay.io/userx/openstack-operator-bundle:v0.6.2,quay.io/userx/openstack-operator-bundle:v0.6.3" \
make catalog-build catalog-push
```

**Pros:** Complete control, builds full upgrade path from scratch, reproducible
**Cons:** Must list all bundles manually

**Recommendation:** Use strategy #3 for production releases and strategy #1 for iterative development.

## Deploying the Update

After building and pushing:

1. Update the CatalogSource to use the new catalog image:
   ```bash
   oc patch catalogsource openstack-operator-index \
     -n openstack-operators \
     --type merge \
     -p '{"spec":{"image":"quay.io/userx/openstack-operator-index:v0.6.2"}}'
   ```

2. Delete the catalog pod to force refresh:
   ```bash
   oc delete pod -n openstack-operators -l olm.catalogSource=openstack-operator-index
   ```

3. Check for upgrade:
   ```bash
   oc get subscription openstack -n openstack-operators
   ```

4. Approve the InstallPlan (if using Manual approval):
   ```bash
   oc patch installplan <installplan-name> \
     -n openstack-operators \
     --type merge \
     -p '{"spec":{"approved":true}}'
   ```

## Troubleshooting

### Upgrade not detected

Check that both versions are in the catalog:
```bash
oc get packagemanifest openstack-operator -n openstack-operators -o json | \
  jq -r '.status.channels[] | select(.name=="alpha") | .entries[] | .name'
```

Both the current and new version should appear.

### Bundle image not found

Ensure the previous bundle was pushed:
```bash
skopeo inspect docker://quay.io/userx/openstack-operator-bundle:v0.6.1
```

## Complete Example Workflows

### Approach A: Incremental Builds (Using CATALOG_BASE_IMG)

Building v0.6.1, v0.6.2, and v0.6.3 incrementally:

```bash
# First custom version (from upstream v0.6.0)
IMAGENAMESPACE=userx IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.0 VERSION=0.6.1 \
IMG=quay.io/userx/openstack-operator:v0.6.1 \
make manifests generate bindata build docker-build docker-push bundle bundle-build bundle-push catalog-build catalog-push

# Second version (builds on v0.6.1 catalog)
IMAGENAMESPACE=userx IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.1 VERSION=0.6.2 \
IMG=quay.io/userx/openstack-operator:v0.6.2 \
PREV_BUNDLE_IMG=quay.io/userx/openstack-operator-bundle:v0.6.1 \
CATALOG_BASE_IMG=quay.io/userx/openstack-operator-index:v0.6.1 \
make manifests generate bindata build docker-build docker-push bundle bundle-build bundle-push catalog-build catalog-push

# Third version (builds on v0.6.2 catalog)
IMAGENAMESPACE=userx IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.2 VERSION=0.6.3 \
IMG=quay.io/userx/openstack-operator:v0.6.3 \
PREV_BUNDLE_IMG=quay.io/userx/openstack-operator-bundle:v0.6.2 \
CATALOG_BASE_IMG=quay.io/userx/openstack-operator-index:v0.6.2 \
make manifests generate bindata build docker-build docker-push bundle bundle-build bundle-push catalog-build catalog-push
```

Each catalog contains the full upgrade path from v0.6.0.

### Approach B: Full From Scratch (Using CATALOG_BUNDLE_IMGS)

Building v0.6.3 with complete upgrade path in one step:

```bash
# Build all versions
IMAGENAMESPACE=userx IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.0 VERSION=0.6.1 \
IMG=quay.io/userx/openstack-operator:v0.6.1 \
make manifests generate bindata build docker-build docker-push bundle bundle-build bundle-push

IMAGENAMESPACE=userx IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.1 VERSION=0.6.2 \
IMG=quay.io/userx/openstack-operator:v0.6.2 \
make manifests generate bindata build docker-build docker-push bundle bundle-build bundle-push

IMAGENAMESPACE=userx IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
REPLACES=openstack-operator.v0.6.2 VERSION=0.6.3 \
IMG=quay.io/userx/openstack-operator:v0.6.3 \
make manifests generate bindata build docker-build docker-push bundle bundle-build bundle-push

# Build complete catalog with all bundles
IMAGENAMESPACE=userx IMAGE_TAG_BASE=quay.io/userx/openstack-operator \
VERSION=0.6.3 IMG=quay.io/userx/openstack-operator:v0.6.3 \
CATALOG_BUNDLE_IMGS="quay.io/openstack-k8s-operators/openstack-operator-bundle:latest,quay.io/userx/openstack-operator-bundle:v0.6.1,quay.io/userx/openstack-operator-bundle:v0.6.2,quay.io/userx/openstack-operator-bundle:v0.6.3" \
make catalog-build catalog-push
```

This approach is reproducible and doesn't depend on previous catalog images.
