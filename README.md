# Magento Commerce Infrastructure

Kubernetes/ArgoCD deployment for the Magento commerce platform.

## Prerequisites

- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [podman](https://podman.io/getting-started/installation)
- Built `localhost/ricky/magento-web:latest` image (see [magento-commerce-magento](https://github.com/rickyaut/magento-commerce-magento))

## First-time setup

### 1. Start minikube and install ArgoCD

```bash
minikube start
minikube addons enable ingress

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Load the magento image into minikube

```bash
podman save localhost/ricky/magento-web:latest | minikube image load --overwrite=true -
```

> This pipes the image directly into minikube's containerd — no registry needed.

### 3. Create hostPath directories and fix permissions

```bash
minikube ssh "sudo mkdir -p /data/magento/mysqldata /data/magento/opensearch && sudo chmod -R 777 /data/magento"
```

### 4. Add magento.local to /etc/hosts

```bash
echo "127.0.0.1 magento.local" | sudo tee -a /etc/hosts
```

### 5. Deploy via ArgoCD

```bash
kubectl apply -f argocd/magento-commerce-magento.yaml
```

ArgoCD will sync the Helm chart from this repo and deploy Magento, MariaDB, and OpenSearch.

### 6. Expose via port-forward

```bash
sudo kubectl port-forward svc/magento 80:80 -n magento-commerce
```

Access at **http://magento.local**

- Admin: http://magento.local/admin (username: `admin`, password: `admin123`)

---

## After a computer restart

minikube and the loaded image do not persist across restarts.

```bash
# 1. Start minikube
minikube start

# 2. Reload the image
podman save localhost/ricky/magento-web:latest | minikube image load --overwrite=true -

# 3. Restart magento pod to use the reloaded image
kubectl rollout restart deployment magento -n magento-commerce

# 4. Port-forward
sudo kubectl port-forward svc/magento 80:80 -n magento-commerce
```

ArgoCD and all Kubernetes resources persist — no need to re-apply manifests.

---

## Repository structure

```
argocd/
  magento-commerce-magento.yaml   # ArgoCD Application manifest
helm/
  magento-commerce-magento/       # Helm chart for Magento + MariaDB + OpenSearch
podman/
  compose.yaml                    # Local dev dependencies (postgres, redis)
```

## Helm chart values

Key values in `helm/magento-commerce-magento/values.yaml`:

| Key | Default | Description |
|-----|---------|-------------|
| `image.repository` | `localhost/ricky/magento-web` | Magento image |
| `image.pullPolicy` | `Never` | Use local minikube image |
| `ingress.host` | `magento.local` | Hostname |
| `mariadb.hostPath` | `/data/magento/mysqldata` | MariaDB data directory on minikube node |
| `opensearch.hostPath` | `/data/magento/opensearch` | OpenSearch data directory on minikube node |
