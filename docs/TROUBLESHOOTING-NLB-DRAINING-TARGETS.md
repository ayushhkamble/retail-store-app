# Troubleshooting: NLB Target Groups Draining — Ingress Domain Not Accessible

## Summary

After deploying the retail-store-app on EKS using Terraform, the domain name exposed by the NGINX Ingress Controller was not reachable in the browser. Checking the AWS Console showed that the NLB (Network Load Balancer) target groups were in a **Draining** state, meaning AWS had deregistered the targets because health checks were continuously failing.

This document explains the root cause, all the things that were wrong, and the exact fixes applied.

---

## Environment

| Component         | Detail                                      |
|-------------------|---------------------------------------------|
| EKS Mode          | EKS Auto Mode (`cluster_compute_config`)    |
| Ingress           | NGINX Ingress Controller (Helm via Blueprints Addons) |
| Load Balancer     | AWS NLB (Network Load Balancer)             |
| IaC               | Terraform + `aws-ia/eks-blueprints-addons`  |
| Kubernetes Version | 1.33                                       |

---

## Symptom

- The NLB hostname from `kubectl get svc -n ingress-nginx ingress-nginx-controller` was resolving in DNS but returning no response / timing out.
- Opening the URL in a browser showed the site was unreachable.
- In the AWS Console → EC2 → Target Groups, the registered targets were showing status: **Draining**.

---

## Root Cause Analysis

There were **three compounding issues**, all stemming from a fundamental incompatibility between **EKS Auto Mode** and the original configuration.

---

### Issue 1: Wrong NLB Target Type — `instance` instead of `ip`

**Original config:**
```hcl
{
  name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-nlb-target-type"
  value = "instance"
}
```

**Why this failed:**

With `instance` target type, the NLB registers the EC2 node's IP and routes traffic to a NodePort. This works fine with standard EKS managed node groups where EC2 instances are visible and directly addressable.

However, **EKS Auto Mode** abstracts node management away. Nodes provisioned by Auto Mode are not registered in the same way as traditional EC2 instances — the in-tree cloud provider cannot reliably map them into NLB target groups using the `instance` model. As a result, the NLB registered the instances but health checks never passed, and AWS began draining and deregistering them.

**Fix:** Changed target type to `ip`, which routes directly to the NGINX controller pod's IP address inside the VPC. This works regardless of the underlying node provisioning model.

```hcl
{
  name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-nlb-target-type"
  value = "ip"
}
```

---

### Issue 2: Hardcoded Health Check Port `10254` Was Unreachable by the NLB

**Original config:**
```hcl
{
  name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-health-check-port"
  value = "10254"
}
```

**Why this failed:**

Port `10254` is NGINX's internal metrics/healthz port. It is exposed on the pod but is **not part of the LoadBalancer service spec** — it's not a named service port, and it's not reachable by the NLB directly.

With `instance` target type, the NLB pings nodes on the NodePort — port `10254` was never a NodePort, so health checks failed immediately. With `ip` target type, the NLB pings pod IPs directly, but port `10254` still needs to be exposed and accessible, and using a hardcoded port creates fragility.

**Fix:** Changed the health check port to `traffic-port`, which is a special AWS keyword that tells the NLB to use the same port as the actual service traffic. This always points to the correct reachable port.

```hcl
{
  name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-health-check-port"
  value = "traffic-port"
}
```

The health check path `/healthz` was kept as-is — NGINX does respond to `/healthz` on its traffic ports.

---

### Issue 3: `externalTrafficPolicy: Local` With `instance` Target Type Caused Traffic Drops

**Original config:**
```hcl
{
  name  = "controller.service.externalTrafficPolicy"
  value = "Local"
}
```

**Why this was problematic:**

`externalTrafficPolicy: Local` tells `kube-proxy` to only forward traffic to pods running on the **same node** that received the traffic. This is great for preserving client source IPs, but it has a critical side effect:

> If the NLB routes a request to a node that does **not** have an NGINX controller pod running on it, `kube-proxy` drops the packet. The NLB then marks that target as unhealthy.

Combined with `instance` target type (which registers all nodes, not just nodes with NGINX pods), this creates a situation where many targets are unreachable for most requests.

**Fix:** Changed to `externalTrafficPolicy: Cluster`, which allows `kube-proxy` to forward traffic across nodes to wherever an NGINX pod is running. Traffic always reaches the pod, at the cost of an extra hop.

```hcl
{
  name  = "controller.service.externalTrafficPolicy"
  value = "Cluster"
}
```

> **Note:** If you need to preserve the original client IP (e.g., for rate limiting or geo-blocking in NGINX), you can switch back to `Local` but you must also ensure the NLB only registers nodes where NGINX pods are running. This can be done with pod topology spread constraints or by using the `ip` target type (which registers pods, not nodes) combined with `Local` policy.

---

### Issue 4: AWS Load Balancer Controller Was Disabled

**Original config:**
```hcl
# enable_aws_load_balancer_controller = true   <-- commented out
```

**Why this mattered:**

EKS Auto Mode is designed to work with the **AWS Load Balancer Controller**, not the legacy in-tree Kubernetes cloud provider. The in-tree provider uses older AWS APIs and does not understand the newer NLB behaviors required by Auto Mode. Without the AWS LB Controller:

- Annotations like `nlb-target-type: ip` may be ignored or mishandled
- The NLB provisioning falls back to the in-tree provider, which lacks support for `ip` mode on Auto Mode nodes
- Health check configurations are not applied correctly

**Fix:** Enabled the AWS Load Balancer Controller as an addon:

```hcl
enable_aws_load_balancer_controller = true
aws_load_balancer_controller = {
  most_recent = true
  namespace   = "kube-system"
}
```

---

## The Fix — Final `addons.tf`

```hcl
module "eks_addons" {
  source  = "aws-ia/eks-blueprints-addons/aws"
  version = "~> 1.0"

  cluster_name      = module.retail_app_eks.cluster_name
  cluster_endpoint  = module.retail_app_eks.cluster_endpoint
  cluster_version   = module.retail_app_eks.cluster_version
  oidc_provider_arn = module.retail_app_eks.oidc_provider_arn

  enable_cert_manager = true
  cert_manager = {
    most_recent = true
    namespace   = "cert-manager"
  }

  # Required for EKS Auto Mode
  enable_aws_load_balancer_controller = true
  aws_load_balancer_controller = {
    most_recent = true
    namespace   = "kube-system"
  }

  enable_ingress_nginx = true
  ingress_nginx = {
    most_recent = true
    namespace   = "ingress-nginx"

    set = [
      { name = "controller.service.type",               value = "LoadBalancer" },
      { name = "controller.service.externalTrafficPolicy", value = "Cluster" }, # changed from Local
      { name = "controller.resources.requests.cpu",     value = "100m" },
      { name = "controller.resources.requests.memory",  value = "128Mi" },
      { name = "controller.resources.limits.cpu",       value = "200m" },
      { name = "controller.resources.limits.memory",    value = "256Mi" }
    ]

    set_sensitive = [
      {
        name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-scheme"
        value = "internet-facing"
      },
      {
        name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-type"
        value = "nlb"
      },
      {
        # Changed from "instance" to "ip" — required for EKS Auto Mode
        name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-nlb-target-type"
        value = "ip"
      },
      {
        name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-health-check-path"
        value = "/healthz"
      },
      {
        # Changed from hardcoded "10254" to "traffic-port"
        name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-health-check-port"
        value = "traffic-port"
      },
      {
        name  = "controller.service.annotations.service\\.beta\\.kubernetes\\.io/aws-load-balancer-health-check-protocol"
        value = "HTTP"
      }
    ]
  }

  depends_on = [module.retail_app_eks]
}
```

---

## Steps to Apply the Fix

```bash
# 1. Plan to review changes
terraform plan

# 2. Apply the configuration
terraform apply

# 3. If the old NLB still has draining targets, force the service to recreate
#    (Helm will provision a fresh NLB with correct settings)
kubectl delete svc ingress-nginx-controller -n ingress-nginx

# 4. Wait for the new NLB to come up and pass health checks (~2-3 minutes)
kubectl get svc -n ingress-nginx -w

# 5. Verify targets are healthy in AWS Console:
#    EC2 → Load Balancers → find your NLB → Target Groups → check status = healthy

# 6. Get the new domain name
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

## Quick Reference: What Changed

| Setting | Before (broken) | After (fixed) | Why |
|---|---|---|---|
| `nlb-target-type` | `instance` | `ip` | EKS Auto Mode doesn't support instance target registration |
| `health-check-port` | `10254` | `traffic-port` | Port 10254 is not reachable by the NLB; traffic-port uses the actual service port |
| `externalTrafficPolicy` | `Local` | `Cluster` | `Local` drops traffic on nodes without NGINX pods |
| AWS LB Controller | disabled | enabled | Required for correct NLB behavior on EKS Auto Mode |

---

## Key Concept: Why EKS Auto Mode Changes Things

Standard EKS gives you managed node groups — regular EC2 instances you can see in the console, with standard ENIs and NodePorts. The in-tree Kubernetes cloud provider was built for this model.

EKS Auto Mode abstracts this away. It provisions and manages nodes on your behalf, but those nodes are not always visible or addressable in the same way. The **AWS Load Balancer Controller** was built to work at the pod/IP level (`ip` target type), which bypasses the node layer entirely and is the supported path for Auto Mode.

**Rule of thumb:** If you're using EKS Auto Mode, always use:
- AWS Load Balancer Controller (not the in-tree provider)
- `ip` target type (not `instance`)
- `externalTrafficPolicy: Cluster` unless you have a specific reason to use `Local`

---

*Resolved on: July 5, 2026*
*Environment: EKS Auto Mode, Kubernetes 1.33, AWS NLB, NGINX Ingress Controller*
