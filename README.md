# 🚀 Production Deployment Guide — FluxCD with Kubernetes

A step-by-step guide for bootstrapping FluxCD on a production Kubernetes cluster with Traefik ingress, Prometheus monitoring, and Grafana dashboards.

---

## 📋 Prerequisites

The following must be in place before you begin:

- [ ] A configured Kubernetes cluster
- [ ] `kubectl` installed and configured
- [ ] Flux CLI installed
- [ ] `git` installed
- [ ] GitHub account with access to the repository
- [ ] Docker installed (required to generate bcrypt hash)
- [ ] A domain with the ability to create DNS records

---

## Part 1 — Preparation

### Step 1 — Install Flux CLI

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

### Step 2 — Create a GitHub Personal Access Token

1. Go to **GitHub → Settings → Developer settings**
2. Navigate to **Personal access tokens → Tokens (classic)**
3. Click **Generate new token (classic)**
4. Give it a descriptive name, e.g. `flux-production`
5. Check the **full `repo` scope**
6. Click **Generate token** and copy it immediately

> ⚠️ You will not be able to see the token again after leaving the page.

### Step 3 — Set environment variables

```bash
export GITHUB_USER=<your-github-username>
export GITHUB_REPO=<your-repository-name>
export GITHUB_TOKEN=<your-personal-access-token>
```

### Step 4 — Verify your cluster context

```bash
kubectl config get-contexts         # list all contexts
kubectl config use-context <name>   # switch to the correct context
kubectl config current-context      # confirm your selection
```

---

## Part 2 — Bootstrap FluxCD

### Step 5 — Bootstrap Flux on the production cluster

```bash
flux bootstrap github \
  --context=<context-name> \
  --owner=${GITHUB_USER} \
  --repository=${GITHUB_REPO} \
  --branch=main \
  --personal \
  --path=clusters/production \
  --token-auth
```

### Step 6 — Fetch the latest changes from GitHub

```bash
git pull origin main
```

### Step 7 — Verify that Flux is installed and syncing

```bash
flux get kustomizations
```

You should see `flux-system` with status `True`.

---

## Part 3 — DNS Configuration

### Step 8 — Get Traefik's external IP

```bash
kubectl get svc -n traefik
```

Note the value of `EXTERNAL-IP`.

### Step 9 — Create a DNS A record

In your DNS provider, create an A record:

```
<your-production-name>.cloudist.solutions  →  <EXTERNAL-IP>
```

### Step 10 — Verify DNS resolution

```bash
nslookup <your-production-name>.cloudist.solutions
```

> ⏳ Wait until it responds with the correct IP before proceeding.

---

## Part 4 — Manually Create Secrets

> ⚠️ **Secrets are never committed to Git. They are created manually, directly in Kubernetes.**

### Step 11 — Wait until namespaces are created by Flux

```bash
kubectl get namespaces
```

Wait until `grafana` and `prometheus` appear with status `Active`.

### Step 12 — Create Grafana admin secret

```bash
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='your-password' \
  -n grafana
```

### Step 13 — Generate bcrypt hash for Prometheus

```bash
docker run --rm httpd:2 htpasswd -nbBC 12 admin your-password
```

Save the output securely, for example in a password manager like Keeper.

Example output:

```
admin:$2y$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Step 14 — Create Prometheus basic-auth secret

Copy the full hash (the part **after** `admin:`) and run:

```bash
kubectl create secret generic prometheus-basic-auth \
  --from-literal=users='admin:$2y$12$YOUR_HASH_HERE' \
  -n prometheus
```

Then create the Prometheus exporter-auth secret:

```bash
kubectl create secret generic prometheus-exporter-auth \
  --from-literal=username=admin \
  --from-literal=password=YOUR_PASSWORD \
  -n prometheus
```

---

## Part 5 — Verification

### Step 15 — Verify that FluxCD is syncing

```bash
flux get kustomizations
```

All entries should show `Ready: True`.

### Step 16 — Verify that pods are running

```bash
kubectl get pods -n prometheus
kubectl get pods -n grafana
```

All pods should show status `Running`.

### Step 17 — Test authentication

```bash
# Should return 401 (unauthorized)
curl -s -o /dev/null -w "%{http_code}" https://YOUR_DOMAIN/prometheus

# Should return 302 (redirect — authenticated)
curl -s -o /dev/null -w "%{http_code}" -u admin:YOUR_PASSWORD https://YOUR_DOMAIN/prometheus
```

---

## ⚠️ Important Reminders

| # | Reminder |
|---|----------|
| 1 | **Secrets are never stored in Git** — always create them manually in the cluster |
| 2 | If rebuilding the environment from scratch, always run Step 12–14 **before** FluxCD syncs the apps |
| 3 | `prometheus-basic-auth` must use the key `users` (not `username`/`password`) for Traefik to work correctly |
