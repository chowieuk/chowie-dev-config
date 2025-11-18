# Chowie.dev K3s Kubernetes Cluster

This repository contains the declarative configuration and manifests for my personal Kubernetes cluster.

## Architecture Overview

*   **Cluster Engine:** [K3s](https://k3s.io/) (Lightweight Kubernetes) running on Linux (Ubuntu 24.04 via Hetzner)
*   **Ingress Controller:** NGINX Ingress Controller (replacing the bundled Traefik).
*   **Load Balancer:** Klipper (K3s built-in ServiceLB).
*   **TLS/SSL:** Cert-Manager with Let's Encrypt.

## Repository Structure

*   `apps/`: Application manifests (Deployments, Services, Ingresses).
    *   `maze-racer/`: A full-stack distributed application.
    *   `astro-frontend/`: Static frontend portfolio site.
    *   `hit-counter-api/`: Backend microservice.
*   `infra/`: Infrastructure configuration and Helm value overrides.

## Infrastructure Setup

To reproduce the cluster state, the following configuration is required.

### 1. K3s Bootstrap
The cluster is initialized with the bundled Traefik ingress controller disabled to favor NGINX.

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --disable=traefik \
  --write-kubeconfig-mode 644
```

### 2. Ingress Controller Deployment
The NGINX Ingress Controller is installed via Helm. It is configured to utilize the K3s built-in ServiceLB (Klipper) via a standard `LoadBalancer` service, ensuring clean separation from the host network.

**Installation:**
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f infra/ingress-nginx-values.yaml
```

### 3. GitOps Engine (Argo CD)
Deployments are managed automatically by Argo CD.

**Prerequisite:**
Before proceeding, ensure the `ghcr-creds` secret exists in the `default` namespace so private images can be pulled.
*(Replace the placeholders with your actual GitHub Personal Access Token)*

```bash
kubectl create secret docker-registry ghcr-creds \
  --docker-server=ghcr.io \
  --docker-username=<YOUR_GITHUB_USERNAME> \
  --docker-password=<YOUR_PAT_TOKEN> \
  --docker-email=<YOUR_EMAIL>
```

**1. Install Argo CD:**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**2. Wait for Argo CD to start:**
```bash
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

**3. Apply the Bootstrap App:**
This connects the cluster to this repository and triggers the deployment of all applications in the `apps/` directory.
```bash
kubectl apply -f infra/bootstrap.yaml
```
