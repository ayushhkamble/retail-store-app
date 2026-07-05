# Retail Store App — Microservices Guide

> A complete walkthrough of the application architecture, every microservice, how they talk to each other, and the infrastructure that runs it all.

---

## Table of Contents

- [What Is This App?](#what-is-this-app)
- [Big Picture Architecture](#big-picture-architecture)
- [The Microservices](#the-microservices)
  - [UI Service](#1-ui-service)
  - [Catalog Service](#2-catalog-service)
  - [Cart Service](#3-cart-service)
  - [Checkout Service](#4-checkout-service)
  - [Orders Service](#5-orders-service)
- [How Services Talk to Each Other](#how-services-talk-to-each-other)
- [Data Stores](#data-stores)
- [Infrastructure Layer](#infrastructure-layer)
- [GitOps with ArgoCD](#gitops-with-argocd)
- [Helm Charts](#helm-charts)
- [Deployment Flow](#deployment-flow)

---

## What Is This App?

The Retail Store is a deliberately over-engineered sample e-commerce application. It is not trying to be simple — it is trying to show you what a real-world microservices system looks like when deployed on AWS with modern DevOps practices.

You get a working online store (browse products, add to cart, checkout, view orders) built across **5 independent services**, each with its own language, its own database, and its own deployment lifecycle. Everything runs on **Amazon EKS with Auto Mode**, managed through **GitOps via ArgoCD**, and provisioned entirely with **Terraform**.

---

## Big Picture Architecture

```
                        ┌─────────────────────────────────────┐
                        │           User's Browser             │
                        └──────────────┬──────────────────────┘
                                       │  HTTP
                                       ▼
                        ┌─────────────────────────────────────┐
                        │         NGINX Ingress Controller     │
                        │         (AWS NLB — internet-facing)  │
                        └──────────────┬──────────────────────┘
                                       │
                                       ▼
                        ┌─────────────────────────────────────┐
                        │             UI Service               │
                        │         Java · Spring Boot           │
                        │       (Server-side rendered HTML)    │
                        └────┬──────────┬──────────┬──────────┘
                             │          │          │          │
                    ┌────────▼─┐  ┌─────▼──┐  ┌───▼────┐  ┌─▼──────┐
                    │ Catalog  │  │  Cart  │  │Checkout│  │ Orders │
                    │    Go    │  │  Java  │  │Node.js │  │  Java  │
                    └────┬─────┘  └───┬────┘  └───┬────┘  └───┬────┘
                         │            │            │            │
                       MySQL      DynamoDB       Redis      PostgreSQL
                                                    │
                                               ┌────▼────┐
                                               │  Orders │ ← also calls on submit
                                               │  (SQS / │
                                               │ RabbitMQ│
                                               └─────────┘
```

The **UI service is the only service exposed externally**. Everything else lives on the internal cluster network and is only reachable from within the VPC.

---

## The Microservices

### 1. UI Service

| | |
|---|---|
| **Language** | Java 21 |
| **Framework** | Spring Boot + Thymeleaf |
| **Role** | Frontend + BFF (Backend for Frontend) |
| **Port** | 8080 |

The UI is the face of the entire application. It renders HTML pages on the server using Thymeleaf templates and acts as the orchestrator that talks to all four backend services on behalf of the browser.

At build time, the UI auto-generates strongly typed HTTP clients from the OpenAPI specs of Catalog, Cart, Checkout, and Orders using the **Kiota** code generation plugin. This means the UI never makes raw HTTP calls — it uses generated client classes, which keeps things type-safe and always in sync with the API contracts.

**What it does:**
- Renders the product listing page by calling Catalog
- Renders the shopping cart by calling Cart
- Drives the checkout flow through the Checkout service
- Shows order history from the Orders service
- Hosts an AI-powered chat assistant via **Spring AI** integrated with **AWS Bedrock Converse** and OpenAI

**Services it calls:**
```
http://catalog:80   → browse products
http://carts:80     → manage shopping cart
http://checkout:80  → checkout flow
http://orders:80    → order history
```

---

### 2. Catalog Service

| | |
|---|---|
| **Language** | Go |
| **Framework** | Gin |
| **Role** | Product catalog API |
| **Database** | MySQL (prod) · SQLite (local dev) |
| **Port** | 8080 |

The Catalog service is a pure data API. It holds all product information — names, descriptions, prices, images, tags — and serves it to whoever asks (in this case, the UI).

It is stateless in terms of request handling; all state lives in MySQL. For local development, it falls back to SQLite so you don't need a database running.

**API endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/catalog/products` | List products with tag filter, pagination, and ordering |
| `GET` | `/catalog/products/{id}` | Get a single product by ID |
| `GET` | `/catalog/tags` | List all available product tags |
| `GET` | `/catalog/size` | Get total product count (used for pagination) |

**Services it calls:** None — this is a leaf service.

---

### 3. Cart Service

| | |
|---|---|
| **Language** | Java 21 |
| **Framework** | Spring Boot |
| **Role** | Shopping cart management |
| **Database** | Amazon DynamoDB (or in-memory for local dev) |
| **Port** | 8080 |

The Cart service manages every customer's shopping cart. Each cart is stored as a collection of items keyed by `customerId`. The storage backend is configurable — in production it uses DynamoDB for its high availability and low-latency key-value access, and locally you can run it in-memory without any AWS dependency.

One neat feature is cart merging — when a guest user adds items and then logs in, their guest cart can be merged into their user cart via the `/merge` endpoint.

**API endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/carts/{customerId}` | Retrieve the full cart |
| `DELETE` | `/carts/{customerId}` | Clear and delete the cart |
| `GET` | `/carts/{customerId}/items` | List all items in cart |
| `POST` | `/carts/{customerId}/items` | Add a new item |
| `PATCH` | `/carts/{customerId}/items` | Update item quantity |
| `DELETE` | `/carts/{customerId}/items/{itemId}` | Remove a specific item |
| `GET` | `/carts/{customerId}/merge?sessionId=` | Merge guest cart into user cart |

**Chaos endpoints** (built-in for resilience testing):

| Method | Path | What it does |
|--------|------|-------------|
| `POST` | `/chaos/status/{code}` | Force all API responses to return a specific HTTP status code |
| `POST` | `/chaos/latency/{delay}` | Add artificial delay (ms) to all responses |
| `POST` | `/chaos/health` | Force health checks to fail |

These are incredibly useful for testing how the rest of the system behaves when Cart is slow or returning errors.

**Services it calls:** None — this is a leaf service.

---

### 4. Checkout Service

| | |
|---|---|
| **Language** | TypeScript (Node.js) |
| **Framework** | NestJS |
| **Role** | Checkout session orchestration |
| **Database** | Redis (session storage) |
| **Port** | 8000 |

The Checkout service is the most interesting orchestrator in the backend. It manages the multi-step checkout process: collecting shipping details, calculating shipping rates, computing taxes, and finally placing the order.

Each customer's in-progress checkout is stored as a serialized session in Redis. This makes it fast to read and update during the checkout flow, and it's automatically cleared once the order is submitted.

**The checkout flow step by step:**

```
1. User clicks "Checkout"
   → UI calls POST /checkout/{customerId}/update
   → Checkout stores items + shipping address in Redis
   → Calls Shipping service to get available shipping rates
   → Returns calculated total (subtotal + tax + shipping)

2. User selects shipping option and confirms
   → UI calls POST /checkout/{customerId}/submit
   → Checkout reads session from Redis
   → Calls Orders service to create the persisted order
   → Deletes the Redis session
   → Returns order confirmation with ID
```

**API endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/checkout/{customerId}` | Get current checkout state |
| `POST` | `/checkout/{customerId}/update` | Update shipping details and recalculate totals |
| `POST` | `/checkout/{customerId}/submit` | Place the order and clear the session |

**Services it calls:**
- `Orders` — to create the final order on submit
- `Shipping` — to fetch available shipping rates

---

### 5. Orders Service

| | |
|---|---|
| **Language** | Java 21 |
| **Framework** | Spring Boot |
| **Role** | Order persistence and event publishing |
| **Database** | PostgreSQL |
| **Messaging** | AWS SQS · RabbitMQ · in-memory (configurable) |
| **Port** | 8080 |

The Orders service is where orders go to live permanently. When Checkout calls it, it persists the order to PostgreSQL and then publishes an event to a message queue for any downstream consumers (warehouse systems, notification services, etc.).

The messaging backend is completely swappable via environment variables — use in-memory for local dev, RabbitMQ for self-hosted setups, or SQS for production on AWS.

**API endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/orders` | List all orders |
| `POST` | `/orders` | Create a new order (called by Checkout) |

**Services it calls:** None — it is called by Checkout, not the other way around.

---

## How Services Talk to Each Other

All communication between services is **synchronous HTTP (REST)**. There is no service mesh. The service discovery is simple Kubernetes DNS — each service name resolves to its ClusterIP.

```
UI          → catalog:80    (product browsing)
UI          → carts:80      (cart management)
UI          → checkout:80   (checkout flow)
UI          → orders:80     (order history)
Checkout    → orders:80     (place order on submit)
```

The wiring is defined in the umbrella Helm chart values:

```yaml
# src/app/chart/values.yaml
ui:
  app:
    endpoints:
      catalog: http://catalog:80
      carts: http://carts:80
      checkout: http://checkout:80
      orders: http://orders:80

checkout:
  retail:
    checkout:
      endpoints:
        orders: http://orders:80
```

---

## Data Stores

Each service owns its own database — there is no shared database between services. This is a core microservices principle: each service is the single source of truth for its own data.

| Service | Store | Why this store? |
|---------|-------|-----------------|
| Catalog | MySQL / SQLite | Relational queries for product filtering and tagging |
| Cart | Amazon DynamoDB | High-speed key-value access by `customerId`; scales automatically |
| Checkout | Redis | Ephemeral session data; fast read/write; auto-expiry |
| Orders | PostgreSQL | Durable, relational order records; ACID compliance |
| Orders | SQS / RabbitMQ | Async event publishing for downstream systems |

---

## Infrastructure Layer

All AWS infrastructure is managed by Terraform under the `terraform/` directory.

### What Terraform Creates

**Networking (VPC)**
- Custom VPC with CIDR `10.0.0.0/16`
- 3 public subnets + 3 private subnets across 3 Availability Zones
- NAT Gateway (single, configurable) for private subnet outbound traffic
- Internet Gateway for public subnet traffic
- Subnet tags for EKS load balancer discovery

**EKS Cluster**
- EKS **Auto Mode** enabled — AWS manages node provisioning automatically
- Kubernetes version 1.33
- Private node placement (nodes in private subnets)
- KMS encryption for secrets
- Public + private API endpoint access

**Cluster Add-ons**
- **AWS Load Balancer Controller** — required for NLB `ip` target type in Auto Mode
- **NGINX Ingress Controller** — routes external traffic to services
- **Cert Manager** — handles SSL/TLS certificate management

### Key Terraform Files

| File | Purpose |
|------|---------|
| `main.tf` | VPC and EKS cluster definitions |
| `addons.tf` | Helm-deployed cluster add-ons |
| `argocd.tf` | ArgoCD installation and configuration |
| `variables.tf` | All configurable input variables |
| `locals.tf` | Computed values, tags, subnet CIDRs |
| `outputs.tf` | Useful outputs (cluster name, kubeconfig command, LB URL) |
| `security.tf` | Security groups and IAM roles |

---

## GitOps with ArgoCD

ArgoCD is installed on the cluster by Terraform. It watches the GitHub repository and automatically syncs the desired state from Git into the cluster.

**ArgoCD Project: `retail-store`**
- Source: `https://github.com/LondheShubham153/retail-store-sample-app`
- Target namespace: `retail-store`
- Each microservice has its own ArgoCD Application defined under `argocd/applications/`

**Individual ArgoCD Applications:**

| Application | Helm Chart |
|-------------|-----------|
| `retail-store-ui` | `src/ui/chart` |
| `retail-store-catalog` | `src/catalog/chart` |
| `retail-store-cart` | `src/cart/chart` |
| `retail-store-checkout` | `src/checkout/chart` |
| `retail-store-orders` | `src/orders/chart` |

When you push a change to the repository, ArgoCD detects the drift between Git and the running cluster and syncs it automatically — no manual `kubectl apply` needed.

**Access the ArgoCD UI:**

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d

# Port-forward to the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open in browser
# https://localhost:8080  (admin / <password from above>)
```

---

## Helm Charts

Every service has its own Helm chart under `src/<service>/chart/`. The charts include:

- `Deployment` — pods, resource limits, environment variables
- `Service` — ClusterIP for internal traffic
- `Ingress` — routing rules through NGINX
- `HorizontalPodAutoscaler` — auto-scaling based on CPU/memory
- `PodDisruptionBudget` — ensures availability during node disruptions
- `ConfigMap` — service-specific configuration
- `ServiceAccount` — Kubernetes RBAC identity for each service

The **umbrella chart** at `src/app/chart/` is a parent chart that wraps all five service charts. It sets inter-service endpoint URLs and lets you deploy the entire application with a single `helm install`.

---

## Deployment Flow

Here is the end-to-end flow from a developer pushing code to the application updating in production:

```
Developer pushes code to GitHub
         │
         ▼
GitHub Actions triggers
         │
         ├─► Builds Docker image for changed service
         ├─► Pushes image to Amazon ECR (private)
         └─► Updates the Helm chart image tag in Git
                        │
                        ▼
              ArgoCD detects Git change
                        │
                        ▼
              ArgoCD syncs to EKS cluster
                        │
                        ▼
              Kubernetes rolling update
                        │
                        ▼
              New pods come up, old pods drain
                        │
                        ▼
              Application updated, zero downtime
```

For a simpler deployment without CI/CD (main branch), you skip GitHub Actions and deploy using public ECR images directly via Helm or the ArgoCD UI.

---

## Troubleshooting Reference

If things go wrong, the most common issues and their fixes are documented here:

- **NLB targets draining / domain not opening** → See [`docs/TROUBLESHOOTING-NLB-DRAINING-TARGETS.md`](./TROUBLESHOOTING-NLB-DRAINING-TARGETS.md)
- **Pods stuck in Pending** → Check node capacity: `kubectl describe nodes`
- **ImagePullBackOff** → Check ECR permissions or verify public image tag exists
- **ArgoCD not syncing** → Check repo URL and branch in ArgoCD application spec
- **Cannot connect to cluster** → Re-run `aws eks update-kubeconfig --name <cluster> --region <region>`

---

*Built with care for the TrainWithShubham Community*
