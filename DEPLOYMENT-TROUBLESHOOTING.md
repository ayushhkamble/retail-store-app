# Deployment Troubleshooting Guide

This document covers every error encountered while deploying the Retail Store App on EKS using GitHub Actions, ArgoCD, and Helm. Each section explains what went wrong, why it happened, and exactly how it was fixed.

---

## Table of Contents

1. [GitHub Actions: Docker Build Fails with Exit Code 126](#1-github-actions-docker-build-fails-with-exit-code-126)
2. [kubectl: Command Not Found](#2-kubectl-command-not-found)
3. [kubectl Still Not Found After Installation](#3-kubectl-still-not-found-after-installation)
4. [ArgoCD Apps Showing Unknown Status](#4-argocd-apps-showing-unknown-status)
5. [ArgoCD Target Revision Pointing to Wrong Branch](#5-argocd-target-revision-pointing-to-wrong-branch)
6. [ArgoCD Port-Forward Connection Refused](#6-argocd-port-forward-connection-refused)
7. [GitHub Actions: Helm values.yaml Update Fails — Image Tag Not Written](#7-github-actions-helm-valuesyaml-update-fails--image-tag-not-written)
8. [GitHub Actions: Push Rejected Due to Concurrent Jobs Writing to gitops Branch](#8-github-actions-push-rejected-due-to-concurrent-jobs-writing-to-gitops-branch)

---

## 1. GitHub Actions: Docker Build Fails with Exit Code 126

### Error Message
```
ERROR: failed to build: failed to solve: process "/bin/sh -c ./mvnw dependency:go-offline -B -q"
did not complete successfully: exit code: 126
```

### Affected Services
- `ui` (Deploy ui job)
- `cart` (Deploy cart job)
- `orders` (Deploy orders job)

### Why It Happened
Exit code `126` on Linux means the file exists but is **not executable**. The `mvnw` (Maven Wrapper) script for these three Java services was committed from a Windows machine. On Windows, the concept of file execute permissions doesn't exist, so git stored the file mode as `100644` (non-executable) instead of `100755` (executable).

When the GitHub Actions runner (Linux) tried to run `./mvnw` inside the Docker build container, the OS refused because the file lacked the execute bit.

The `checkout` and `catalog` services were not affected because they use Node.js and Go respectively — neither uses `mvnw`.

### How to Verify
```bash
git ls-files --stage src/ui/mvnw src/cart/mvnw src/orders/mvnw
```
Output showing `100644` confirms the missing execute bit:
```
100644 19529ddf8c6eaa08c5c75ff80652d21ce4b72f8c 0   src/cart/mvnw
100644 19529ddf8c6eaa08c5c75ff80652d21ce4b72f8c 0   src/orders/mvnw
100644 19529ddf8c6eaa08c5c75ff80652d21ce4b72f8c 0   src/ui/mvnw
```

### Fix
Use `git update-index` to set the executable bit directly in git's index without needing a Linux machine:

```bash
git update-index --chmod=+x src/ui/mvnw src/cart/mvnw src/orders/mvnw
```

Verify the mode changed to `100755`:
```bash
git ls-files --stage src/ui/mvnw src/cart/mvnw src/orders/mvnw
# Should now show 100755 for all three
```

Commit and push:
```bash
git commit -m "fix: set executable bit on mvnw for ui, cart, and orders"
git push origin gitops
```

---

## 2. kubectl: Command Not Found

### Error Message
```
bash: kubectl: command not found
```

### Why It Happened
`kubectl` was not installed on the machine. It is not bundled with AWS CLI, Docker, or any other tool — it must be installed separately.

### Fix
Install kubectl using winget on Windows:

```bash
winget install -e --id Kubernetes.kubectl
```

After installation, connect kubectl to your EKS cluster by updating the kubeconfig:

```bash
# Get the exact command from terraform output
cd terraform
terraform output configure_kubectl

# Run the command it gives you, e.g.:
aws eks update-kubeconfig --region us-east-1 --name retail-store-xxxx
```

---

## 3. kubectl Still Not Found After Installation

### Error Message
```
bash: kubectl: command not found
```
(Even after winget reported successful installation)

### Why It Happened
The terminal session (Git Bash inside VS Code) was opened **before** kubectl was installed. The `PATH` environment variable is loaded when the shell starts, so it didn't include the new kubectl installation path. The winget installer itself even warns:

```
Path environment variable modified; restart your shell to use the new value.
```

Additionally, Git Bash does not automatically pick up PATH changes made by Windows installers the same way CMD or PowerShell does.

### How to Verify
```bash
where.exe kubectl
# Returns: INFO: Could not find files for the given pattern(s).
```

kubectl was actually installed at:
```
C:\Users\Lenovo\AppData\Local\Microsoft\WinGet\Packages\Kubernetes.kubectl_Microsoft.Winget.Source_8wekyb3d8bbwe\kubectl.exe
```
But this path was not in the shell's PATH.

### Fix

**Option 1 — Temporary fix for the current session:**
```bash
export PATH="$PATH:/c/Users/Lenovo/AppData/Local/Microsoft/WinGet/Packages/Kubernetes.kubectl_Microsoft.Winget.Source_8wekyb3d8bbwe"
```

**Option 2 — Permanent fix (add to Git Bash profile):**
```bash
echo 'export PATH="$PATH:/c/Users/Lenovo/AppData/Local/Microsoft/WinGet/Packages/Kubernetes.kubectl_Microsoft.Winget.Source_8wekyb3d8bbwe"' >> ~/.bashrc
source ~/.bashrc
```

Verify:
```bash
kubectl version --client
```

---

## 4. ArgoCD Apps Showing Unknown Status

### Symptom
All 5 ArgoCD applications showed `Unknown / Unknown` for both sync and health status after being applied to the cluster.

### Why It Happened
ArgoCD could not connect to the GitHub repository. The apps were referencing `https://github.com/ayushhkamble/retail-store-app` but no repository credentials had been configured in ArgoCD. Without credentials, ArgoCD cannot clone the repo to compare the desired state.

### Fix
Add the repository to ArgoCD with authentication credentials:

1. In the ArgoCD UI go to **Settings → Repositories → Connect Repo**
2. Select **VIA HTTPS**
3. Fill in:
   - **Type**: `git`
   - **Project**: `retail-store`
   - **Repository URL**: `https://github.com/ayushhkamble/retail-store-app`
   - **Username**: your GitHub username
   - **Password**: a GitHub Personal Access Token (PAT)

To generate a GitHub PAT:
1. GitHub → Profile → **Settings**
2. **Developer settings** → **Personal access tokens** → **Tokens (classic)**
3. Click **Generate new token (classic)**
4. Add a note (e.g. `argocd`), set expiration, check the **`repo`** scope
5. Click **Generate token** and copy it immediately

After connecting the repo, click **REFRESH APPS** in ArgoCD. All apps will transition from Unknown to Synced/Healthy.

---

## 5. ArgoCD Target Revision Pointing to Wrong Branch

### Symptom
All 5 ArgoCD applications had `Target Revision: main` but the CI/CD pipeline was pushing Helm chart changes to the `gitops` branch. ArgoCD was syncing from `main` which didn't have the updated image tags.

Additionally, the `repoURL` in all application manifests was pointing to the original tutorial repository (`LondheShubham153/retail-store-sample-app`) instead of the forked repo.

### Fix
Update all 5 ArgoCD application YAML files under `argocd/applications/`:

```yaml
# Before
source:
  repoURL: https://github.com/LondheShubham153/retail-store-sample-app
  targetRevision: main

# After
source:
  repoURL: https://github.com/ayushhkamble/retail-store-app
  targetRevision: gitops
```

Files updated:
- `argocd/applications/retail-store-cart.yaml`
- `argocd/applications/retail-store-catalog.yaml`
- `argocd/applications/retail-store-checkout.yaml`
- `argocd/applications/retail-store-orders.yaml`
- `argocd/applications/retail-store-ui.yaml`

Commit, push, and re-apply:
```bash
git add argocd/applications/
git commit -m "fix: update ArgoCD apps to use gitops branch and correct repoURL"
git push origin gitops
kubectl apply -f argocd/applications/ -n argocd
```

---

## 6. ArgoCD Port-Forward Connection Refused

### Error Message
```
error forwarding port 8080 -> 8080: error forwarding port in network namespace:
dial tcp 127.0.0.1:8080: connect: connection refused
error: lost connection to pod
```

### Why It Happened
The ArgoCD server pod was not yet fully ready when the port-forward was initiated. The pod was either still starting up or had briefly restarted after the application sync triggered a rolling update.

### Fix
Wait for the ArgoCD server pod to be fully running before port-forwarding:

```bash
kubectl get pods -n argocd
```

Wait until `argocd-server-*` shows `Running` with `1/1` ready, then re-run:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

If connection is still refused, try port 80 instead:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

Or disable TLS on the ArgoCD server for simpler local access:
```bash
kubectl patch deployment argocd-server -n argocd \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--insecure"}]'

kubectl port-forward svc/argocd-server -n argocd 8080:80
```

---

## 7. GitHub Actions: Helm values.yaml Update Fails — Image Tag Not Written

### Error Message
```
❌ Update failed - restoring backup
Error: Process completed with exit code 1
```

### Why It Happened
The `deploy.yml` workflow uses an `awk` script to update only the **top-level** `image:` block in `src/<service>/chart/values.yaml` — the one that controls the main service container. It specifically avoids overwriting infrastructure images like `mysql`, `redis`, `dynamodb-local`, etc.

The `awk` script works by looking for a line that starts exactly with `^image:` (no leading whitespace, at the root level of the YAML). If the `values.yaml` was accidentally reformatted, indented, or the `image:` key was nested under another key, the pattern would never match. After the `awk` run, the workflow validates:

```bash
if grep -q "${ECR_REPO}" "${VALUES_FILE}" && grep -q "\"${TAG}\"" "${VALUES_FILE}"; then
```

If either the ECR repo URL or the new tag is not found in the file, it restores the backup and exits with code 1.

This happened when the `values.yaml` for a service had its `image:` block indented or restructured during a manual edit, breaking the `awk` pattern match.

### How to Verify
Open the affected service's `values.yaml` and check the structure:

```yaml
# Correct — image: at root level, no leading whitespace
image:
  repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/retail-store-ui
  tag: "abc1234"
  pullPolicy: IfNotPresent

# Broken — image: indented or nested, awk pattern won't match
service:
  image:
    repository: ...
    tag: "abc1234"
```

### Fix
Ensure the `image:` block is at the **root level** of `values.yaml` with no leading whitespace:

```yaml
image:
  repository: ""
  tag: "latest"
  pullPolicy: IfNotPresent
```

After fixing the structure, re-trigger the workflow either by pushing a change or using **workflow_dispatch** from the GitHub Actions UI. The `awk` script will now match the `image:` key correctly and update the repository and tag.

---

## 8. GitHub Actions: Push Rejected Due to Concurrent Jobs Writing to gitops Branch

### Error Message
```
⚠️ Push failed for cart, attempt 1/3. Retrying...
⚠️ Push failed for ui, attempt 1/3. Retrying...
❌ Failed to push orders after 3 attempts
Error: Process completed with exit code 1
```

### Why It Happened
The `deploy.yml` workflow uses a matrix strategy to build and deploy all changed services **in parallel**:

```yaml
strategy:
  matrix: ${{ fromJson(needs.detect-changes.outputs.matrix) }}
  fail-fast: false
```

Each parallel job (ui, cart, orders, etc.) independently:
1. Checks out the `gitops` branch
2. Updates its own `src/<service>/chart/values.yaml`
3. Commits the change
4. Tries to push to `origin gitops`

When multiple jobs finish at nearly the same time, they all try to push to the same branch simultaneously. The first job to push succeeds. Every other job gets rejected because their local branch is now behind the remote — git refuses a non-fast-forward push.

The workflow has a built-in retry with rebase to handle this:

```bash
for i in {1..3}; do
  if git push origin gitops; then
    break
  else
    git pull --rebase origin gitops
    sleep 2
  fi
done
```

However, if more than 3 services are pushing at the same time and all retries collide within the sleep window, the retry logic can still be exhausted — especially if the runner is slow or network latency is high.

### Fix
The retry + rebase logic in the workflow handles this in most cases. If it still fails after 3 attempts, re-trigger the failed jobs individually using **"Re-run failed jobs"** in the GitHub Actions UI. Since only one job runs at a time on re-run, the push conflict won't occur.

For a permanent fix, serialize the commit step by adding a concurrency group to the workflow:

```yaml
jobs:
  deploy:
    concurrency:
      group: helm-commit-${{ github.ref }}
      cancel-in-progress: false
```

This ensures only one job at a time can commit and push to the `gitops` branch, eliminating the race condition entirely while still allowing Docker builds to run in parallel.

---

## Full Deployment Flow (After All Fixes)

```
Push code to gitops branch
        ↓
GitHub Actions triggers deploy.yml
        ↓
Builds Docker image → pushes to ECR
        ↓
Updates image tag in src/<service>/chart/values.yaml
        ↓
Commits updated values.yaml back to gitops branch
        ↓
ArgoCD detects change in gitops branch (polls every 3 min)
        ↓
ArgoCD syncs Helm chart → rolling update on EKS
        ↓
New version live on the cluster
```

Any code push to the `gitops` branch will automatically propagate to the EKS cluster within a few minutes with no manual steps required.
