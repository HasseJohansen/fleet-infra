# MetalLB CRD Installation Fix

## Problem
MetalLB CRDs were not being installed because Flux was applying resources in alphabetical order without respecting dependencies.

## Root Causes

### 1. Incorrect API Version
The `apps/networking/metallb/config/pool.yaml` file used `apiVersion: metallb.io/v1` but MetalLB Helm chart 0.16.1 only provides CRDs with `v1beta1` version. The `v1` API version does not exist in this chart version.

**Fix**: Changed back to `metallb.io/v1beta1`.

### 2. Missing Kustomization Structure
The `apps/` directory lacked proper `kustomization.yaml` files at various levels, causing Flux to treat the entire directory as raw manifests and apply files alphabetically. This meant:
- `pool.yaml` (IPAddressPool custom resource) was applied BEFORE the HelmRelease installed the CRDs
- Health checks for monitoring and samba-operator resources also failed, blocking the entire `apps` kustomization

**Fix**: Added `kustomization.yaml` files at each directory level to ensure proper kustomize builds with correct dependency ordering.

## Future Work: Restructure Flux Code

**IMPORTANT**: The current structure has a dependency ordering problem that should be fixed:

- The `apps` kustomization includes many subdirectories (controllers, media, monitoring, networking, policy, share, storage)
- Resources are applied in alphabetical order of directory names
- If any resource in an earlier directory (like `monitoring`) fails due to missing CRDs, it blocks ALL subsequent directories (including `networking/metallb`)

**Recommended restructuring**:
1. Split the `apps` kustomization into separate top-level kustomizations per functional area
2. Use explicit `dependsOn` between kustomizations to control ordering
3. Ensure each functional area (monitoring, networking, storage, etc.) has its own independent kustomization that can succeed/fail independently
4. This would prevent a missing CRD in monitoring from blocking MetalLB installation

**Current blockers**:
- `monitoring` needs AlertmanagerConfig CRD (from Prometheus Operator Helm chart)
- `networking/samba-operator` needs SmbCommonConfig CRD (from Samba Operator Helm chart)
- These should not block `networking/metallb` which has its own dependency chain

## Current Status
- ✅ MetalLB CRDs installed and working
- ✅ IPAddressPool and L2Advertisement resources applied successfully
- ✅ MetalLB pods running
- ⚠️ monitoring blocked by missing AlertmanagerConfig CRD
- ⚠️ samba-operator blocked by missing SmbCommonConfig CRD
