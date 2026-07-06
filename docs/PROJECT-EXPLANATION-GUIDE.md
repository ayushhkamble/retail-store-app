# Retail Store App — Complete Project Explanation Guide

This guide explains the entire project end-to-end: architecture, tech stack, infrastructure, CI/CD pipeline, and GitOps flow. Written to help you explain this project confidently in interviews or technical discussions.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Microservices Breakdown](#3-microservices-breakdown)
4. [Infrastructure — Terraform](#4-infrastructure--terraform)
5. [Containerization — Dockerfiles](#5-containerization--dockerfiles)
6. [Helm Charts](#6-helm-charts)
7. [ArgoCD — GitOps](#7-argocd--gitops)
8. [CI/CD Pipeline — GitHub Actions](#8-cicd-pipeline--github-actions)
9. [End-to-End Flow](#9-end-to-end-flow)
10. [Key Interview Questions & Answers](#10-key-interview-questions--answers)

---

## 1. Project Overview

The Retail Store App is a **cloud-native, microservices-based e-commerce application** deployed on AWS EKS (Elastic Kubernetes Service). It demonstrates a production-grade DevOps setup using:

| Concern | Tool |
|---|---|
| Cloud Infrastructure | AWS (EKS, VPC, ECR) |
| Infrastructure as Code | Terraform |
| Containerization | Docker |
| Container Orchestration | Kubernetes (EKS Auto Mode) |
| Package Management | Helm |
| GitOps / CD | ArgoCD |
| CI Pipeline | GitHub Actions |
| Container Registry | Amazon ECR |
| Branching Strategy | `gitops` branch as the delivery branch |

The application is split into **5 independent microservices**, each with its own codebase, Docker image, Helm chart, and ArgoCD application manifest.

---

## 2. Architecture Diagram

```
                        ┌──────────────────────────────┐
                        │         Developer             │
                        │   pushes to gitops branch     │
                        └──────────────┬───────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────┐
                        │      GitHub Actions           │
                        │  (detect → build → push ECR  │
                        │   → update Helm values.yaml) │
                        └──────────────┬───────────────┘
                                       │ commits values.yaml
                                       ▼
                        ┌──────────────────────────────┐
                        │    GitHub (gitops branch)    │
                        │  src/<service>/chart/        │
                        │       values.yaml            │
                        └──────────────┬───────────────┘
                                       │ ArgoCD polls (every 3 min)
                                       ▼
┌────────────────────────────────────────────────────────────────┐
│                        AWS EKS Cluster                         │
│                                                                │
│   ┌───────────┐    ┌──────────┐    ┌──────────────────────┐   │
│   │  ArgoCD   │───▶│  Helm    │───▶│   retail-store NS    │   │
│   │ (argocd   │    │  Sync    │    │                      │   │
│   │    NS)    │    └──────────┘    │  ui → catalog        │   │
│   └───────────┘                   │  cart → checkout      │   │
│                                   │  orders               │   │
│                                   └──────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
         ▲
         │ terraform apply
┌────────────────┐
│   Terraform    │
│ (VPC + EKS +  │
│  ArgoCD setup) │
└────────────────┘
```

---

## 3. Microservices Breakdown

Each service is independently deployable. Here is what each one does and the technology it uses:

### UI Service
- **Language**: Java 21 (Spring Boot)
- **Role**: Frontend — serves the web interface to the user's browser
- **Port**: 8080 internally, exposed via NGINX Ingress externally
- **Talks to**: catalog, cart, checkout, orders (all via internal ClusterIP DNS)
- **Persistence**: Stateless — no database

### Catalog Service
- **Language**: Go
- **Role**: Product catalog — serves product listings, categories, and details
- **Port**: 8080
- **Persistence**: In-memory by default, MySQL when stateful mode is enabled

### Cart Service
- **Language**: Java 21 (Spring Boot)
- **Role**: Shopping cart — stores items a user has added before checkout
- **Port**: 8080
- **Persistence**: In-memory by default, DynamoDB when stateful mode is enabled

### Checkout Service
- **Language**: Node.js
- **Role**: Handles the checkout process — calculates totals, initiates order creation
- **Port**: 8080
- **Persistence**: In-memory by default, Redis when stateful mode is enabled
- **Talks to**: orders service

### Orders Service
- **Language**: Java 21 (Spring Boot)
- **Role**: Order management — stores and tracks placed orders
- **Port**: 8080
- **Persistence**: In-memory by default, PostgreSQL + RabbitMQ when stateful mode is enabled

### Service Communication

All services communicate over Kubernetes internal DNS. No service is directly exposed to the internet except UI through the NGINX Ingress Controller.

```
User Browser
    │
    ▼
NGINX Ingress (LoadBalancer — NLB on AWS)
    │
    ▼
UI Service (port 80)
    ├──▶ Catalog Service  (http://catalog:80)
    ├──▶ Cart Service     (http://carts:80)
    ├──▶ Checkout Service (http://checkout:80)
    └──▶ Orders Service   (http://orders:80)

Checkout Service
    └──▶ Orders Service   (http://orders:80)
```

---

## 4. Infrastructure — Terraform

All AWS infrastructure is provisioned using Terraform, located in the `terraform/` directory.

### What Terraform Creates

#### VPC (`main.tf`)
```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # Creates:
  # - VPC with CIDR 10.0.0.0/16
  # - Public subnets  (for Load Balancers)
  # - Private subnets (for EKS worker nodes)
  # - NAT Gateway     (allows private nodes to reach internet)
  # - Internet Gateway
}
```

Public subnets are tagged for AWS Load Balancer Controller to create external NLBs. Private subnets are tagged for internal load balancers.

#### EKS Cluster (`main.tf`)
```hcl
module "retail_app_eks" {
  source = "terraform-aws-modules/eks/aws"
  # Creates:
  # - EKS control plane
  # - EKS Auto Mode (no manually managed node groups)
  # - node_pools = ["general-purpose"]
  # - Public + private endpoint access
  # - KMS encryption for cluster secrets
}
```

**EKS Auto Mode** is used — AWS manages the node lifecycle, scaling, and patching automatically. No EC2 node groups need to be configured manually.

#### ArgoCD Installation (`argocd.tf`)
```hcl
resource "helm_release" "argocd" {
  # Installs ArgoCD via the official Helm chart into the argocd namespace
  # Configured with --insecure flag for port-forward access
  # ClusterIP service (not exposed publicly)
}

resource "kubectl_manifest" "argocd_projects" {
  # Applies argocd/projects/*.yaml after ArgoCD is installed
}

resource "kubectl_manifest" "argocd_apps" {
  # Applies argocd/applications/*.yaml after projects are created
}
```

Terraform uses `depends_on` to ensure proper ordering: VPC → EKS → ArgoCD Helm install → ArgoCD Projects → ArgoCD Applications.

#### Key Outputs (`outputs.tf`)
After `terraform apply`, these commands are printed:

```bash
# Connect kubectl to the cluster
aws eks update-kubeconfig --region <region> --name <cluster-name>

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d

# Get the app URL
echo 'http://'$(kubectl get svc -n ingress-nginx \
  ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

---

## 5. Containerization — Dockerfiles

Each service uses a **multi-stage Docker build** to keep the final image small and secure.

### Java Services (ui, cart, orders) — Pattern

```dockerfile
# Stage 1: BUILD
FROM public.ecr.aws/amazonlinux/amazonlinux:2023 AS build-env
RUN dnf install -y maven java-21-amazon-corretto-headless
COPY .mvn .mvn
COPY mvnw .
COPY pom.xml .
RUN ./mvnw dependency:go-offline -B -q   # Download all deps (cacheable layer)
COPY ./src ./src
RUN ./mvnw -DskipTests package -q        # Compile and package JAR

# Stage 2: RUNTIME
FROM public.ecr.aws/amazonlinux/amazonlinux:2023
RUN dnf install -y java-21-amazon-corretto-headless
# Non-root user for security
RUN useradd --home "/app" --create-home --user-group --uid 1000 appuser
COPY --from=build-env /app.jar .         # Only copy the JAR, not build tools
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

Key points:
- **Amazon Linux 2023** base image — AWS-optimized, minimal attack surface
- **Multi-stage build** — build tools (Maven, JDK) are discarded; only the JAR goes into the final image
- **Non-root user (UID 1000)** — security best practice; pod security context enforces this
- **dependency:go-offline** cached as a separate layer — speeds up rebuilds when only source code changes

### Go Service (catalog)

```dockerfile
# Stage 1: BUILD
FROM public.ecr.aws/docker/library/golang AS build-env
WORKDIR /go/src/app
COPY go.mod go.sum ./
RUN go mod download          # Download deps
COPY . .
RUN go build -o /retail-store-catalog main.go

# Stage 2: RUNTIME
FROM public.ecr.aws/amazonlinux/amazonlinux:2023
COPY --from=build-env /retail-store-catalog .
EXPOSE 8080
ENTRYPOINT ["/retail-store-catalog"]
```

Go produces a **single static binary** — the runtime image needs no runtime installed at all.

### Node.js Service (checkout)

```dockerfile
# Stage 1: BUILD
FROM public.ecr.aws/docker/library/node:20 AS build-env
WORKDIR /app
COPY package*.json ./
RUN npm ci                   # Install deps
COPY . .
RUN npm run build            # Compile TypeScript/assets

# Stage 2: RUNTIME
FROM public.ecr.aws/docker/library/node:20-alpine
COPY --from=build-env /app/dist ./dist
COPY --from=build-env /app/node_modules ./node_modules
EXPOSE 8080
CMD ["node", "dist/index.js"]
```

---

## 6. Helm Charts

Each microservice has its own Helm chart at `src/<service>/chart/`. There is also a parent **umbrella chart** at `src/app/chart/` that can deploy all services together as one unit.

### Per-Service Chart Structure

```
src/<service>/chart/
├── Chart.yaml           # Chart metadata (name, version)
├── values.yaml          # Default configuration values
└── templates/
    ├── deployment.yaml  # Kubernetes Deployment
    ├── service.yaml     # ClusterIP Service
    ├── configmap.yaml   # App configuration (env vars, endpoints)
    ├── serviceaccount.yaml
    ├── hpa.yaml         # HorizontalPodAutoscaler (disabled by default)
    └── pdb.yaml         # PodDisruptionBudget (disabled by default)
```

### values.yaml — Key Sections Explained

```yaml
# The Docker image to deploy — GitHub Actions updates these two fields on every push
image:
  repository: <account>.dkr.ecr.<region>.amazonaws.com/retail-store-ui
  tag: "abc1234"       # ← 7-char git SHA, updated by CI
  pullPolicy: Always

# Resource limits — prevents one pod from starving the node
resources:
  limits:
    memory: 512Mi
  requests:
    cpu: 128m
    memory: 512Mi

# Security hardening applied at the pod level
securityContext:
  capabilities:
    drop: [ALL]           # Drop all Linux capabilities
  readOnlyRootFilesystem: true   # Filesystem is immutable
  runAsNonRoot: true
  runAsUser: 1000

# Persistence configuration — in-memory by default, real DB when enabled
app:
  persistence:
    provider: in-memory   # Switch to 'dynamodb', 'mysql', 'postgres', 'redis'

# Prometheus metrics scraping annotations
metrics:
  enabled: true
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
```

### Stateful vs Stateless Modes

Each chart supports **two persistence modes**:

| Mode | Config | What runs |
|---|---|---|
| Stateless (default) | `provider: in-memory` | Service only, no database pods |
| Stateful | `provider: mysql/dynamodb/redis/postgres` + `create: true` | Service + sidecar database pod |

The umbrella chart's `values-stateful.yaml` enables all databases at once:
```yaml
cart:
  app.persistence.provider: dynamodb
  dynamodb.create: true
catalog:
  app.persistence.provider: mysql
  mysql.create: true
checkout:
  app.persistence.provider: redis
  redis.create: true
orders:
  app.persistence.provider: postgres + rabbitmq
  postgresql.create: true
  rabbitmq.create: true
```

### Umbrella Chart (`src/app/chart/Chart.yaml`)

```yaml
name: retail-store-sample-chart
dependencies:
  - name: retail-store-sample-cart-chart
    alias: cart
    repository: file://../../cart/chart
  - name: retail-store-sample-catalog-chart
    alias: catalog
    ...
```

This allows deploying the entire application with a single `helm install` command. Each dependency is a local file reference (`file://`) pointing to the individual service charts.

---

## 7. ArgoCD — GitOps

ArgoCD implements the **GitOps** pattern — the Git repository is the single source of truth for what should be running in the cluster.

### ArgoCD Project (`argocd/projects/retail-store-project.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: retail-store
spec:
  # Which repos ArgoCD is allowed to sync from
  sourceRepos:
    - 'https://github.com/ayushhkamble/retail-store-app'

  # Where ArgoCD is allowed to deploy
  destinations:
    - namespace: retail-store
      server: https://kubernetes.default.svc   # in-cluster

  # Which Kubernetes resource types ArgoCD can create
  clusterResourceWhitelist:
    - kind: Namespace
    - kind: ClusterRole
    - kind: ClusterRoleBinding
  namespaceResourceWhitelist:
    - kind: Deployment, Service, ConfigMap, Secret, HPA, PDB, Ingress ...
```

The **AppProject** acts as a security boundary — it restricts what repos, namespaces, and resource types an application inside the project can touch.

### ArgoCD Application (`argocd/applications/retail-store-<service>.yaml`)

All 5 applications follow the same structure:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: retail-store-cart
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # Deploy order (1 = backends first, 2 = ui)
spec:
  project: retail-store

  source:
    repoURL: https://github.com/ayushhkamble/retail-store-app
    targetRevision: gitops          # Watch the gitops branch
    path: src/cart/chart            # Helm chart location in the repo
    helm:
      valueFiles:
        - values.yaml               # Use this values file for configuration

  destination:
    server: https://kubernetes.default.svc
    namespace: retail-store

  syncPolicy:
    automated:
      prune: true       # Delete resources removed from Git
      selfHeal: true    # Revert any manual kubectl changes
    syncOptions:
      - CreateNamespace=true   # Create retail-store namespace if missing
```

**Sync Waves** control deployment order:
- Wave 1: cart, catalog, checkout, orders (backend services)
- Wave 2: ui (frontend — starts after all backends are up)

**selfHeal: true** means if someone manually edits a Kubernetes resource with `kubectl`, ArgoCD will automatically revert it back to what's in Git within 3 minutes. Git is always the source of truth.

**prune: true** means if you delete a file from Git, ArgoCD will delete the corresponding Kubernetes resource. No orphaned resources.

---

## 8. CI/CD Pipeline — GitHub Actions

The pipeline lives at `.github/workflows/deploy.yml` and has 3 jobs.

### Trigger

```yaml
on:
  push:
    branches: [gitops]
    paths: ['src/**']     # Only triggers when source code changes
  workflow_dispatch:      # Manual trigger from GitHub UI
```

### Job 1: Detect Changed Services

Uses `git diff HEAD~1 HEAD` to check which `src/<service>/` directories changed. Builds a dynamic matrix — only changed services are built, not all 5 every time.

```bash
for service in ui catalog cart checkout orders; do
  if git diff --name-only HEAD~1 HEAD | grep -q "^src/$service/"; then
    CHANGED_SERVICES+=("$service")
  fi
done
# Output: matrix={"service":["cart","ui"]}
```

This saves build time and ECR storage — if you only changed `src/ui/`, only the `ui` image is rebuilt.

### Job 2: Deploy (matrix — runs in parallel per service)

Each changed service goes through these steps:

**Step 1 — Configure AWS credentials**
```yaml
uses: aws-actions/configure-aws-credentials@v4
# Uses AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY from GitHub Secrets
```

**Step 2 — Login to ECR**
```yaml
uses: aws-actions/amazon-ecr-login@v2
# Authenticates Docker to push to your private ECR registry
```

**Step 3 — Build and push Docker image**
```bash
TAG=$(echo $GITHUB_SHA | cut -c1-7)   # e.g. "abc1234"
ECR_REPO="<account>.dkr.ecr.<region>.amazonaws.com/retail-store-${SERVICE}"

# Create ECR repo if it doesn't exist yet
aws ecr create-repository --repository-name "retail-store-${SERVICE}" ...

docker build -t "${ECR_REPO}:${TAG}" -t "${ECR_REPO}:latest" "src/${SERVICE}/"
docker push "${ECR_REPO}:${TAG}"
docker push "${ECR_REPO}:latest"
```

**Step 4 — Update Helm values.yaml**
```bash
# Uses awk to update only the top-level image block
# Preserves infrastructure images (mysql, redis, dynamodb, etc.)
awk -v repo="${ECR_REPO}" -v tag="${TAG}" '
  /^image:/ { in_main_image = 1 }
  in_main_image && /repository:/ { print "  repository: " repo; next }
  in_main_image && /tag:/ { print "  tag: \"" tag "\""; next }
  { print }
' values.yaml
```

**Step 5 — Commit and push values.yaml back to gitops branch**
```bash
git commit -m "Update ${SERVICE} Helm chart to ${TAG}"
git push origin gitops
# Includes retry logic with rebase for concurrent job push conflicts
```

### Job 3: Deployment Summary

Writes a markdown summary to the GitHub Actions run page showing what was built, what ECR images were pushed, and confirmation that ArgoCD will sync.

### GitHub Secrets Required

| Secret | What it holds |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `AWS_REGION` | e.g. `us-east-1` |
| `AWS_ACCOUNT_ID` | 12-digit AWS account number |

---

## 9. End-to-End Flow

Here is the complete journey from a code change to production:

```
1. Developer edits src/ui/src/main/java/... and pushes to gitops branch

2. GitHub Actions: detect-changes job
   └── Detects only src/ui/ changed → matrix = ["ui"]

3. GitHub Actions: deploy job (for ui only)
   ├── Authenticate to AWS ECR
   ├── docker build src/ui/ → image tag = "7a3f1c2"
   ├── docker push → ECR: retail-store-ui:7a3f1c2
   ├── Update src/ui/chart/values.yaml:
   │     image.repository = <account>.dkr.ecr.us-east-1.amazonaws.com/retail-store-ui
   │     image.tag = "7a3f1c2"
   └── git commit + push to gitops branch

4. ArgoCD (running in EKS) polls GitHub every 3 minutes
   └── Detects src/ui/chart/values.yaml changed on gitops branch

5. ArgoCD syncs retail-store-ui Application
   ├── Runs: helm upgrade retail-store-ui ./src/ui/chart -f values.yaml
   └── Kubernetes performs rolling update:
       ├── Starts new pod with image retail-store-ui:7a3f1c2
       ├── Waits for new pod to pass readiness probe
       └── Terminates old pod

6. New version is live. Zero downtime deployment.
```

---

## 10. Key Interview Questions & Answers

**Q: What is GitOps and how is it implemented here?**

GitOps is a practice where Git is the single source of truth for the desired state of infrastructure and applications. Any change to the cluster must go through a Git commit. Here, ArgoCD watches the `gitops` branch — when `values.yaml` is updated with a new image tag by CI, ArgoCD detects the drift between the Git state and cluster state and syncs automatically.

**Q: What is the difference between CI and CD in this project?**

CI (GitHub Actions) handles building the Docker image, testing, and pushing to ECR. It also updates the Helm chart values in Git. CD (ArgoCD) handles deploying to Kubernetes. They are intentionally separated — CI writes to Git, CD reads from Git. This decoupling means the CD tool never needs AWS credentials for deployment.

**Q: Why do you have a separate `gitops` branch?**

To separate application code commits from deployment state commits. The `gitops` branch contains the authoritative deployment configuration. CI writes back to this branch (image tags), and ArgoCD syncs from it. Using a dedicated branch prevents deployment noise from polluting the main development branch.

**Q: What happens if someone manually changes a Kubernetes resource with kubectl?**

ArgoCD's `selfHeal: true` policy detects the drift within 3 minutes and reverts the change back to what is in Git. This enforces that Git is always the source of truth and prevents configuration drift.

**Q: How does the pipeline handle multiple microservices without rebuilding all of them?**

The `detect-changes` job uses `git diff HEAD~1 HEAD` to check which `src/<service>/` paths changed and builds a dynamic matrix. Only services with actual code changes are built. This saves build time and keeps ECR clean.

**Q: What is a Helm Chart and why is it used here?**

A Helm chart is a package of Kubernetes YAML templates with configurable values. Instead of writing separate YAML files for each service and environment, Helm allows parameterizing the deployment. The `image.tag` in `values.yaml` is the only thing that changes between releases — everything else (resource limits, security context, service ports) stays constant.

**Q: How is the infrastructure provisioned?**

Terraform provisions the AWS infrastructure: a VPC with public/private subnets, an EKS cluster using Auto Mode (no manual node group management), and installs ArgoCD via Helm. After Terraform apply, ArgoCD is running in the cluster and the GitOps loop is active.

**Q: What security practices are applied in this project?**

- Non-root containers (UID 1000) enforced via pod security context
- Read-only root filesystem on all containers
- All Linux capabilities dropped (`drop: [ALL]`)
- Secrets stored in GitHub Secrets (never in code)
- ECR image scanning enabled on push
- ArgoCD AppProject restricts which repos and namespaces applications can use
- IAM least-privilege — CI uses an IAM user with only ECR push + EKS describe permissions

**Q: What is EKS Auto Mode?**

EKS Auto Mode is an AWS feature where AWS manages the EC2 node lifecycle, scaling, patching, and security updates automatically. You define node pools (e.g. `general-purpose`) and AWS handles the rest. This removes the need to manage node groups, launch templates, or cluster autoscaler configuration.

**Q: How do you access ArgoCD?**

ArgoCD is deployed with a ClusterIP service (not publicly exposed). Access is via kubectl port-forward:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Then open `http://localhost:8080`. The initial admin password is retrieved from a Kubernetes secret created by ArgoCD on installation.
