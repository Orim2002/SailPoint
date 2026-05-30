# Linker

A minimal URL shortener built with Flask, deployed on a local [kind](https://kind.sigs.k8s.io/) cluster via Argo CD.

## Prerequisites

- [`kind`](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/)
- [`helm`](https://helm.sh/docs/intro/install/)
- [`argocd` CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) (optional)

## 1. Create the cluster

```bash
kind create cluster --name linker --config kind-config.yaml
```

> The `kind-config.yaml` maps host ports 80 and 443 into the cluster so the ingress controller is reachable from your machine.

## 2. Install the NGINX ingress controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

Add the ingress hostname to `/etc/hosts`:

```bash
echo "127.0.0.1 linker.local" | sudo tee -a /etc/hosts
```

## 3. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --namespace argocd \
  --for=condition=available deployment/argocd-server \
  --timeout=120s
```

Register this repo as an Argo CD application:

```bash
kubectl apply -f argocd/application.yaml
```

Argo CD will watch the `helm/` directory on `main` and sync it into the `linker` namespace automatically, with pruning and self-healing enabled.

Retrieve the initial admin password and access the UI:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

kubectl port-forward svc/argocd-server -n argocd 8443:443
# Open https://localhost:8443  (username: admin)
```

## 4. Verify everything is working

Check sync status:

```bash
kubectl get application linker -n argocd
# SYNC STATUS: Synced   HEALTH: Healthy
```

Check the pod is running:

```bash
kubectl get pods -n linker
```

Hit the app through the ingress:

```bash
curl http://linker.local/health
# {"status": "ok"}

curl -X POST http://linker.local/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'

curl -L http://linker.local/<slug>
```

## Teardown

```bash
kind delete cluster --name linker
```
