# Auto-Deploy Flow: Implementation Plan

## Goal

Automate deployment manifest updates after a successful Docker image release, closing the gap in the GitOps pipeline.

## Current Flow (Manual Gap)

```
simple-go-service-a release workflow
  └─ increment-version → build-and-release → summary
                              │
                              ├─ Binary uploaded to GitHub Release
                              └─ Docker image pushed to ghcr.io
                                       │
                                       ▼
                              *** MANUAL STEP ***
                     Someone updates deployment.yaml with new tag
                                       │
                                       ▼
                              ArgoCD detects change → K8s rollout
```

## Target Flow (Fully Automated)

```
simple-go-service-a release workflow
  └─ increment-version → build-and-release → update-deployment → summary
                                                    │
                                                    ├─ Clones kse-labs-deployment
                                                    ├─ Updates image tag + version labels via yq
                                                    ├─ Commits & pushes to main
                                                    │
                                                    ▼
                                          ArgoCD auto-syncs → K8s rollout
```

## Implementation TODO

### Step 1: Create org-level secret for cross-repo access

- [ ] Create a **fine-grained PAT** scoped to `kse-mseheda/kse-labs-deployment`
  - Permission: `Contents: Read and write`
- [ ] Store as org-level secret: `DEPLOYMENT_REPO_TOKEN`
- [ ] Make it available to `simple-go-service-a` (and future service repos)

### Step 2: Create reusable workflow in trusted-workflows

- [ ] Create `kse-labs-trusted-workflows/.github/workflows/update-deployment-manifest.yaml`
- [ ] Inputs:
  - `deployment-repo` (default: `kse-mseheda/kse-labs-deployment`)
  - `service-name` (required -- used for commit messages AND to derive the deployment path)
  - `image-name` (e.g., `ghcr.io/kse-mseheda/simple-go-service-a`)
  - `new-tag` (e.g., `v0.0.14`)
- [ ] Naming convention: deployment path is derived as `services/<service-name>/deployment.yaml` -- no explicit path input needed
- [ ] Secret: `deployment-repo-token`
- [ ] Steps:
  - Install `yq` if not present (handle macOS ARM + Linux AMD64)
  - Clone deployment repo using token
  - Read current tag, skip if already up-to-date
  - Update 3 YAML fields with `yq`:
    - `.metadata.labels["app.kubernetes.io/version"]`
    - `.spec.template.metadata.labels["app.kubernetes.io/version"]`
    - `.spec.template.spec.containers[0].image`
  - Commit: `deploy(<service>): update image tag <old> -> <new>`
  - Push to main with retry/rebase (handles concurrent releases)
- [ ] Push to `kse-labs-trusted-workflows` main branch

### Step 3: Update service release workflow

- [ ] Add `update-deployment` job to `simple-go-service-a/.github/workflows/release.yaml`:
  ```yaml
  update-deployment:
    needs: [increment-version, build-and-release]
    uses: kse-mseheda/kse-labs-trusted-workflows/.github/workflows/update-deployment-manifest.yaml@main
    secrets:
      deployment-repo-token: ${{ secrets.DEPLOYMENT_REPO_TOKEN }}
    with:
      image-name: ghcr.io/kse-mseheda/simple-go-service-a
      new-tag: ${{ needs.increment-version.outputs.new_tag }}
      service-name: simple-go-service-a
  ```
- [ ] Update `summary` job: add `update-deployment` to `needs` array
- [ ] Add deployment status line to summary output
- [ ] Push to `simple-go-service-a` main branch

### Step 4: Verify end-to-end

- [ ] Trigger a patch release via `workflow_dispatch` on `simple-go-service-a`
- [ ] Verify commit appears in `kse-labs-deployment` with correct image tag
- [ ] Verify version labels updated in `deployment.yaml`
- [ ] Verify ArgoCD syncs the change and pods roll out with new image

## Fields Updated in deployment.yaml

| YAML Path | Before | After |
|---|---|---|
| `metadata.labels["app.kubernetes.io/version"]` | `v0.0.13` | `v0.0.14` |
| `spec.template.metadata.labels["app.kubernetes.io/version"]` | `v0.0.13` | `v0.0.14` |
| `spec.template.spec.containers[0].image` | `ghcr.io/.../simple-go-service-a:v0.0.13` | `ghcr.io/.../simple-go-service-a:v0.0.14` |

## Design Notes

### Service-agnostic reusable workflow

The reusable workflow takes `service-name` and `image-name` as inputs. The deployment path is derived automatically using the naming convention `services/<service-name>/deployment.yaml`:

```yaml
# service-a calls with:
service-name: simple-go-service-a        # -> services/simple-go-service-a/deployment.yaml
image-name: ghcr.io/kse-mseheda/simple-go-service-a

# service-b would call with:
service-name: simple-go-service-b        # -> services/simple-go-service-b/deployment.yaml
image-name: ghcr.io/kse-mseheda/simple-go-service-b
```

**Convention:** The deployment directory in `kse-labs-deployment/services/` must match the service repo name. No explicit path input needed.

### ArgoCD auto-discovery

The `argocd/applicationsets/services.yaml` uses a git directory generator with `path: services/*`. When a new folder `services/<name>/` is created with manifests, ArgoCD automatically creates an Application and syncs it. With `selfHeal: true`, any commit to `kse-labs-deployment/main` triggers an automatic sync -- the moment the workflow pushes the updated tag, ArgoCD picks it up and rolls out new pods.

### Concurrent release safety

If two services release at the same time, both workflows try to push to `kse-labs-deployment/main`. The second push would fail because the remote moved ahead. The retry logic handles this:

1. Push fails (remote has new commit from the other service)
2. `git pull --rebase` pulls the other service's commit
3. Rebase succeeds because they edit different files (e.g., `services/simple-go-service-a/deployment.yaml` vs `services/service-b/deployment.yaml`)
4. Push succeeds on retry (up to 3 attempts)
