# kubernetes# k8s-multi-tenant-hosting

A Kubernetes platform that hosts multiple isolated application tenants on one cluster. Each tenant gets its own namespace, resource limits, secrets, and routin; separated from other tenant(s) on the same infrastructure.

This mirrors how real hosting platforms work at scale - one cluster running multiple environments, each unaware and without access to the others.

---

## What this builds

Two tenants running in full isolation on a local kind cluster

- Separate namespaces per tenant
- Independent deployments with rolling update support
- Resource quotas so no tenant can starve the cluster
- Database credentials injected via Kubernetes Secrets
- Internal service discovery via CoreDNS
- External routing by hostname via Ingress Controller

---

## Architecture
Each tenant functions in its own namespace. Services provide stable internal DNS. Ingress routes external traffic by hostname to the right tenant.

In production on EKS the Ingress Controller would provision an AWS ALB automatically. Security groups on the worker nodes would restrict traffic so only the ALB can reach the pods, and only the pods can reach RDS on port 5432.

---

## Stack

- Kubernetes via kind (local)
- nginx Ingress Controller
- kubectl

Production equivalent: EKS with EC2 node groups or Fargate, AWS Load Balancer Controller, External Secrets Operator pulling from AWS Secrets Manager

---

## Running it locally

```bash
# create the cluster
kind create cluster --name multi-tenant

# install ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# wait for ingress controller
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# apply everything
kubectl apply -f namespaces/
kubectl apply -f secrets/
kubectl apply -f deployments/
kubectl apply -f quotas/
kubectl apply -f services/
kubectl apply -f ingress/
```

---

## Testing routing

```bash
# port forward the ingress controller
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80 &

# hit each tenant
curl -H "Host: tenant-a.local" http://localhost:8080
curl -H "Host: tenant-b.local" http://localhost:8080
```

---

## Checking tenant resource usage

```bash
kubectl describe resourcequota -n tenant-a
kubectl describe resourcequota -n tenant-b
```

This shows live CPU and memory consumption per tenant against their quota — the same view you would use in production to monitor tenant health and decide when to upgrade a customer to a larger tier.

---

## Concepts demonstrated

**Namespace isolation** - tenant-a and tenant-b share the same cluster but cannot see or reach each other by default

**Resource quotas** - each namespace has hard limits on CPU, memory and pod count so one tenant cannot starve another

**Secrets** - DB credentials stored in Kubernetes Secrets and injected as environment variables at pod startup, never hardcoded in the image

**Services** - stable internal DNS so pods can be reached by name regardless of their IP changing

**Ingress** - single entry point routing traffic to the right tenant by hostname
