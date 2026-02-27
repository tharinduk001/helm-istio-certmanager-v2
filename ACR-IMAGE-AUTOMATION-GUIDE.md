# Flux ACR Image Update Automation Guide

This guide documents the complete steps for setting up **automatic container image updates** using Flux with Azure Container Registry (ACR). When a new image is pushed to ACR, Flux will detect it, update the image tag in Git, and roll out the change to the cluster.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Create the ACR Secret (SealedSecret)](#2-create-the-acr-secret-sealedsecret)
3. [Create the ImageRepository](#3-create-the-imagerepository)
4. [Create the ImagePolicy](#4-create-the-imagepolicy)
5. [Add the Image Policy Marker to the Deployment](#5-add-the-image-policy-marker-to-the-deployment)
6. [Create the ImageUpdateAutomation](#6-create-the-imageupdateautomation)
7. [Register All Resources in the Kustomization](#7-register-all-resources-in-the-kustomization)
8. [Commit, Push & Reconcile](#8-commit-push--reconcile)
9. [Verify Everything is Working](#9-verify-everything-is-working)
10. [Error Encountered & Fix](#10-error-encountered--fix)

---

## 1. Prerequisites

- Flux installed with **image automation components** enabled:
  ```bash
  flux bootstrap github \
    --components-extra=image-reflector-controller,image-automation-controller \
    --owner=<GITHUB_USER> \
    --repository=helm-istio-certmanager-v2 \
    --branch=main \
    --path=clusters/k8s-ws-terra-v2 \
    --read-write-key \
    --personal
  ```
  > **Important:** The `--components-extra=image-reflector-controller,image-automation-controller` flag is required. These components are **not installed by default**. If you bootstrapped Flux before without this flag, re-run bootstrap with it (delete the `flux-system` secret first to rotate the deploy key).

- Sealed Secrets controller running in the cluster (for encrypting ACR credentials).
- A working ACR registry (e.g., `k8sworkshopcertd.azurecr.io`).

---

## 2. Create the ACR Secret (SealedSecret)

The `ImageRepository` needs credentials to pull image tags from a **private** ACR registry. We store this as a SealedSecret so it can be safely committed to Git.

### Step 2.1: Create the Docker registry secret (temporary, for sealing)

```bash
kubectl create secret docker-registry acr-secret \
  --namespace=flux-system \
  --docker-server=k8sworkshopcertd.azurecr.io \
  --docker-username=<ACR_USERNAME> \
  --docker-password=<ACR_PASSWORD> \
  --dry-run=client -o yaml > acr-secret.yaml
```

### Step 2.2: Seal the secret with kubeseal

```bash
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=sealed-secrets \
  --scope cluster-wide \
  < acr-secret.yaml \
  > clusters/k8s-ws-terra-v2/flux-system/acr-secret-enc.yaml
```

### Step 2.3: Resulting SealedSecret file

**File:** `clusters/k8s-ws-terra-v2/flux-system/acr-secret-enc.yaml`

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  annotations:
    sealedsecrets.bitnami.com/cluster-wide: "true"
  name: acr-secret
  namespace: flux-system
spec:
  encryptedData:
    .dockerconfigjson: <ENCRYPTED_DATA>
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/cluster-wide: "true"
      name: acr-secret
      namespace: flux-system
    type: kubernetes.io/dockerconfigjson
```

> **Key point:** The namespace must be `flux-system` — the same namespace where the `ImageRepository` lives. The sealed-secrets controller will decrypt this into a real `kubernetes.io/dockerconfigjson` secret named `acr-secret` in `flux-system`.

### Step 2.4: Delete the temporary unencrypted secret

```bash
rm acr-secret.yaml
```

---

## 3. Create the ImageRepository

The `ImageRepository` tells Flux **which container registry to scan** for new tags.

**File:** `clusters/k8s-ws-terra-v2/flux-system/imagerepositories/lms-static-canary.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1
kind: ImageRepository
metadata:
  name: lms-static-canary
  namespace: flux-system
spec:
  secretRef:
    name: acr-secret
  image: k8sworkshopcertd.azurecr.io/lms-static
  interval: 1m
```

| Field | Description |
|-------|-------------|
| `secretRef.name` | References the `acr-secret` in the same namespace for ACR authentication |
| `image` | The full ACR image path (without tag) |
| `interval` | How often Flux scans the registry for new tags (every 1 minute) |

---

## 4. Create the ImagePolicy

The `ImagePolicy` tells Flux **which tag to select** from the scanned tags based on a policy (e.g., semver, alphabetical, numerical).

**File:** `clusters/k8s-ws-terra-v2/flux-system/imagepolicies/lms-static-canary.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1
kind: ImagePolicy
metadata:
  name: lms-static-canary
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: lms-static-canary
  policy:
    semver:
      range: '>=1.0.0'
```

| Field | Description |
|-------|-------------|
| `imageRepositoryRef.name` | Must match the `ImageRepository` name created in Step 3 |
| `policy.semver.range` | Selects the latest tag matching semver `>=1.0.0` (e.g., `v1.0.13`) |

### Other Policy Examples

```yaml
# Patch versions only
policy:
  semver:
    range: '1.0.x'

# Minor and patch versions
policy:
  semver:
    range: '>=1.0.0 <2.0.0'

# Numerical sorting (for build-id based tags like main-abc1234-1611906956)
filterTags:
  pattern: '^main-[a-fA-F0-9]+-(?P<ts>.*)'
  extract: '$ts'
policy:
  numerical:
    order: asc
```

---

## 5. Add the Image Policy Marker to the Deployment

Edit the deployment file and add the **`$imagepolicy` marker comment** on the `image:` line. This tells Flux exactly which line to update when a new tag is resolved.

**File:** `clusters/k8s-ws-terra-v2/lms/deployment-canary.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lms-static-canary
  labels:
    app: lms-static
    version: canary
  namespace: lms
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lms-static
      version: canary
  template:
    metadata:
      labels:
        app: lms-static
        version: canary
    spec:
      containers:
      - name: lms-static-container
        image: k8sworkshopcertd.azurecr.io/lms-static:v1.0.13 # {"$imagepolicy": "flux-system:lms-static-canary"}
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: acr-secret
```

### Marker Format

The marker is an **inline YAML comment** with the format:

```
# {"$imagepolicy": "<namespace>:<imagepolicy-name>"}
```

- `flux-system` — the namespace where the `ImagePolicy` lives
- `lms-static-canary` — the name of the `ImagePolicy` resource

> **Important:** The marker must be on the **same line** as the `image:` field. Flux's setter strategy scans for these markers and replaces the entire image reference (name + tag) with the resolved value from the policy.

### Marker Variants for Custom Resources (e.g., HelmRelease)

```yaml
# Full image (name:tag) — default
image: ghcr.io/org/app:v1.0.0 # {"$imagepolicy": "flux-system:app"}

# Tag only
tag: v1.0.0 # {"$imagepolicy": "flux-system:app:tag"}

# Image name only
repository: ghcr.io/org/app # {"$imagepolicy": "flux-system:app:name"}
```

---

## 6. Create the ImageUpdateAutomation

The `ImageUpdateAutomation` tells Flux **how to commit and push** the image tag changes back to Git.

**File:** `clusters/k8s-ws-terra-v2/flux-system/image-update-automation.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: '{{range .Changed.Changes}}{{print .OldValue}} -> {{println .NewValue}}{{end}}'
    push:
      branch: main
  update:
    path: ./clusters/k8s-ws-terra-v2
    strategy: Setters
```

| Field | Description |
|-------|-------------|
| `sourceRef` | Must reference the `GitRepository` that Flux uses (usually `flux-system`) |
| `git.checkout.ref.branch` | Branch to checkout for reading manifests |
| `git.push.branch` | Branch to push image update commits to |
| `git.commit.author` | Author name/email for the automated commits |
| `git.commit.messageTemplate` | Go template for commit messages (shows old → new image) |
| `update.path` | Directory path to scan for `$imagepolicy` markers |
| `update.strategy` | Must be `Setters` — uses kyaml setters to find and replace markers |

> **Note:** The `update.path` must point to the directory containing your deployment manifests with the `$imagepolicy` markers. In our case `./clusters/k8s-ws-terra-v2` covers all subdirectories including `lms/`.

---

## 7. Register All Resources in the Kustomization

All the image automation resources must be listed in the Flux `kustomization.yaml` so they get applied to the cluster.

**File:** `clusters/k8s-ws-terra-v2/flux-system/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- gotk-components.yaml
- gotk-sync.yaml
- helmrepositories/cert-manager.yaml
- helmrepositories/sealed-secrets.yaml
- imagerepositories/lms-static-canary.yaml    # <-- ImageRepository
- imagepolicies/lms-static-canary.yaml         # <-- ImagePolicy
- image-update-automation.yaml                 # <-- ImageUpdateAutomation
- acr-secret-enc.yaml                         # <-- ACR SealedSecret
```

> **Important:** If you forget to add any of these resources here, they will **not** be applied to the cluster even though the YAML files exist in the repo.

---

## 8. Commit, Push & Reconcile

### Step 8.1: Commit and push all changes

```bash
git add -A
git commit -m "add ACR image update automation"
git push origin main
```

### Step 8.2: Trigger Flux reconciliation (or wait for automatic sync)

```bash
flux reconcile kustomization flux-system --with-source
```

---

## 9. Verify Everything is Working

### Check ImageRepository (is it scanning successfully?)

```bash
flux get image repository lms-static-canary -n flux-system
```

Expected output:
```
NAME                LAST SCAN   SUSPENDED   READY   MESSAGE
lms-static-canary   ...         False       True    successful scan: found 6 tags
```

### Check ImagePolicy (did it resolve a tag?)

```bash
flux get image policy lms-static-canary -n flux-system
```

Expected output:
```
NAME                IMAGE                                           TAG      READY   MESSAGE
lms-static-canary   k8sworkshopcertd.azurecr.io/lms-static         v1.0.13  True    Latest image tag for ... resolved to v1.0.13
```

### Check ImageUpdateAutomation (did it push a commit?)

```bash
flux get image update flux-system -n flux-system
```

Expected output:
```
NAME         READY   STATUS               LAST RUN
flux-system  True    repository up-to-date ...
```

### Check all image resources at once

```bash
flux get images all --all-namespaces
```

### Pull the automated commit and verify

```bash
git pull
grep "image:" clusters/k8s-ws-terra-v2/lms/deployment-canary.yaml
```

Expected output:
```
image: k8sworkshopcertd.azurecr.io/lms-static:v1.0.13 # {"$imagepolicy": "flux-system:lms-static-canary"}
```

---

## 10. Error Encountered & Fix

### The Error

After deploying all the resources, the `ImageRepository` was stuck in a failed state:

```
NAME                READY   STATUS
lms-static-canary   False   failed to configure authentication options: secrets "acr-secret" not found
```

And the `ImagePolicy` was also failing as a result:

```
NAME                READY   STATUS
lms-static-canary   False   retrying in 30s error: no tags in database
```

### Root Cause

This was a **race condition / timing issue**. All resources were deployed at the same time, but:

1. The `acr-secret-enc.yaml` (SealedSecret) was applied to the cluster.
2. The **sealed-secrets controller** needed time to decrypt the SealedSecret into an actual Kubernetes secret (`acr-secret`).
3. Meanwhile, the **image-reflector-controller** (ImageRepository) tried to scan the ACR registry immediately and couldn't find `acr-secret` yet — because the sealed-secrets controller hadn't finished creating it.
4. The ImageRepository entered a retry loop but was slow to recover even after the secret was eventually created.

### The Fix

Once the `acr-secret` was confirmed to exist in the `flux-system` namespace:

```bash
# Verify the secret exists
kubectl get secret acr-secret -n flux-system
```

Force a manual reconciliation of the ImageRepository:

```bash
flux reconcile image repository lms-static-canary -n flux-system
```

After this, the ImageRepository successfully authenticated, scanned the registry, found the tags, the ImagePolicy resolved the latest tag (`v1.0.13`), and the ImageUpdateAutomation committed the updated image tag to Git.

### How to Prevent This

- **Option A:** Deploy the SealedSecret first, wait for the secret to be created, then deploy the image automation resources in a subsequent commit.
- **Option B:** If you hit this error, simply wait for the sealed-secrets controller to create the secret, then run:
  ```bash
  flux reconcile image repository lms-static-canary -n flux-system
  ```
- **Option C:** Use Flux `dependsOn` in your Kustomizations to ensure the sealed-secrets resources are applied before the image automation resources.

---

## File Structure Summary

```
clusters/k8s-ws-terra-v2/
├── flux-system/
│   ├── gotk-components.yaml          # Flux controllers (includes image-reflector & image-automation)
│   ├── gotk-sync.yaml                # GitRepository + Kustomization for flux-system
│   ├── kustomization.yaml            # Lists all resources to apply
│   ├── acr-secret-enc.yaml           # SealedSecret for ACR credentials (flux-system namespace)
│   ├── image-update-automation.yaml  # ImageUpdateAutomation
│   ├── imagerepositories/
│   │   └── lms-static-canary.yaml    # ImageRepository (scans ACR)
│   └── imagepolicies/
│       └── lms-static-canary.yaml    # ImagePolicy (selects semver >=1.0.0)
└── lms/
    └── deployment-canary.yaml        # Deployment with $imagepolicy marker on image line
```

---

## How It Works (End-to-End Flow)

```
1. CI pushes new image → k8sworkshopcertd.azurecr.io/lms-static:v1.0.14
                                    │
2. ImageRepository scans ACR        ▼
   (every 1m)              ┌─────────────────┐
                           │ Found tags:      │
                           │ v1.0.14, v1.0.13 │
                           │ v1.0.12, ...     │
                           └────────┬─────────┘
                                    │
3. ImagePolicy applies              ▼
   semver >=1.0.0          ┌─────────────────┐
                           │ Latest: v1.0.14  │
                           └────────┬─────────┘
                                    │
4. ImageUpdateAutomation            ▼
   (every 5m)              ┌───────────────────────────────┐
                           │ Finds marker in deployment:   │
                           │ # {"$imagepolicy": "..."}     │
                           │ Replaces v1.0.13 → v1.0.14    │
                           │ Commits & pushes to Git       │
                           └────────┬──────────────────────┘
                                    │
5. Flux Kustomization               ▼
   detects new commit      ┌─────────────────┐
   in Git                  │ Applies updated  │
                           │ deployment to    │
                           │ cluster          │
                           └─────────────────┘
```
