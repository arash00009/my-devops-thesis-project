# FluxCD Complete Observability Stack

![FluxCD](https://img.shields.io/badge/FluxCD-GitOps-blue?logo=flux&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-cluster-326CE5?logo=kubernetes&logoColor=white)
![Traefik](https://img.shields.io/badge/Traefik-ingress-EF3B2D?logo=traefikproxy&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-metrics-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-dashboards-F46800?logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-logs-yellow)
![Tempo](https://img.shields.io/badge/Tempo-traces-5941A9)
![License](https://img.shields.io/badge/license-MIT-green)

A production-ready GitOps observability stack bootstrapped with FluxCD. Covers the full three pillars of observability — **metrics**, **logs**, and **traces** — deployed declaratively on Kubernetes with Traefik as the ingress controller and Grafana as the unified frontend.

---

## Table of Contents

- [Overview](#overview)
- [Stack Components](#stack-components)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
  - [Part 1 — Bootstrap & DNS](#part-1--bootstrap--dns)
  - [Part 2 — Secrets](#part-2--secrets)
  - [Part 3–4 — Core Services & Prometheus](#part-34--core-services--prometheus)
  - [Part 5 — Grafana Tempo](#part-5--grafana-tempo)
  - [Part 6–7 — Loki & FluentBit](#part-67--loki--fluentbit)
  - [Part 8 — External Access](#part-8--external-access)
  - [Part 9 — Grafana Dashboards](#part-9--grafana-dashboards)
- [Verification](#verification)
- [External Push Endpoints](#external-push-endpoints)
- [Secrets Reference](#secrets-reference)
- [Known Issues](#known-issues)
- [Contributing](#contributing)

---

## Overview

This repository manages a complete observability stack for Kubernetes using **GitOps principles**. All infrastructure is declared in code and reconciled automatically by FluxCD — no manual `kubectl apply` after initial bootstrap.

The stack is split into nine sequential parts. Parts 1–4 lay the base infrastructure; parts 5–9 build the full observability layer on top.

```
Metrics  →  Prometheus  →  Grafana
Logs     →  FluentBit   →  Loki    →  Grafana
Traces   →  OTLP/HTTP   →  Tempo   →  Grafana
```

---

## Stack Components

| Part | Component | Namespace | Purpose |
|------|-----------|-----------|---------|
| 1 | FluxCD + DNS | `flux-system` | GitOps engine and domain setup |
| 2 | Secrets | multiple | Credentials — never stored in Git |
| 3 | Core Services | `nginx`, `grafana` | Web server and visualization frontend |
| 4 | Prometheus | `prometheus` | Metrics collection and nginx scraping |
| 5 | Grafana Tempo | `tempo` | Distributed tracing via OTLP |
| 6 | Grafana Loki | `loki` | Log aggregation |
| 7 | FluentBit | `fluent-bit` | Log shipping from pods to Loki |
| 8 | External Access | all | Secured HTTPS push endpoints |
| 9 | Grafana Dashboards | `grafana` | Auto-provisioned via ConfigMaps |

---

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │              Kubernetes Cluster          │
                        │                                         │
  External Systems ─────┼──► Traefik (Ingress + TLS)             │
                        │       │                                 │
                        │       ├──► /prometheus  ──► Prometheus  │
                        │       ├──► /grafana     ──► Grafana     │
                        │       ├──► /tempo/ready ──► Tempo       │
                        │       ├──► /push        ──► Pushgateway │
                        │       ├──► /loki/api    ──► Loki        │
                        │       └──► /v1/traces   ──► Tempo       │
                        │                                         │
                        │  FluentBit ──► Loki ──────────► Grafana │
                        │  Prometheus ─────────────────► Grafana  │
                        │  Tempo ──────────────────────► Grafana  │
                        └─────────────────────────────────────────┘
                                         ▲
                                    FluxCD (Git → Cluster)
```

---

## Prerequisites

- A running Kubernetes cluster with a `kubectl` context configured
- [Flux CLI](https://fluxcd.io/docs/installation/) installed locally
- A GitHub account with a Personal Access Token (repo scope)
- A domain with DNS you control
- Traefik deployed and exposing an external IP

---

## Getting Started

> **Follow parts in order** when setting up from scratch. Each part builds on the previous one.

### Part 1 — Bootstrap & DNS

**Install the Flux CLI:**

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

**Set environment variables:**

```bash
export GITHUB_USER=<your-github-username>
export GITHUB_REPO=<your-repository-name>
export GITHUB_TOKEN=<your-personal-access-token>
```

**Bootstrap FluxCD** (run for each cluster):

```bash
# Production
flux bootstrap github \
  --context=<context-name> \
  --owner=${GITHUB_USER} \
  --repository=${GITHUB_REPO} \
  --branch=main \
  --personal \
  --path=clusters/production \
  --token-auth

# Staging
flux bootstrap github \
  --context=<context-name> \
  --owner=${GITHUB_USER} \
  --repository=${GITHUB_REPO} \
  --branch=main \
  --personal \
  --path=clusters/staging \
  --token-auth
```

**Configure DNS:**

```bash
kubectl get svc -n traefik              # Grab the EXTERNAL-IP
# Create an A record: <domain>.example.com → EXTERNAL-IP
nslookup <domain>.example.com          # Verify propagation
```

---

### Part 2 — Secrets

> ⚠️ **All secrets must be created manually in each cluster. Never commit credentials to Git.**

**Grafana admin credentials:**

```bash
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='YOUR_PASSWORD' \
  -n grafana
```

**Prometheus basic auth** (generate bcrypt hash first):

```bash
docker run --rm httpd:2 htpasswd -nbBC 12 admin YOUR_PASSWORD

kubectl create secret generic prometheus-basic-auth \
  --from-literal=users='admin:$2y$12$YOUR_HASH' \
  -n prometheus

kubectl create secret generic prometheus-exporter-auth \
  --from-literal=username=admin \
  --from-literal=password=YOUR_PASSWORD \
  -n prometheus
```

**External push secrets:**

```bash
docker run --rm httpd:2 htpasswd -nbBC 12 external YOUR_EXT_PASSWORD

kubectl create secret generic prometheus-push-auth \
  --from-literal=users='external:$2y$12$YOUR_HASH' \
  -n prometheus

kubectl create secret generic loki-push-auth \
  --from-literal=users='external:$2y$12$YOUR_HASH' \
  -n loki

kubectl create secret generic tempo-push-auth \
  --from-literal=users='external:$2y$12$YOUR_HASH' \
  -n tempo
```

---

### Part 3–4 — Core Services & Prometheus

**Verify all kustomizations are reconciled:**

```bash
flux get kustomizations
kubectl get pods -n prometheus
kubectl get pods -n grafana
```

**Enable nginx metrics** by adding the following to `apps/base/nginx/release.yaml`:

```yaml
values:
  metrics:
    enabled: true
    podAnnotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "9113"
      prometheus.io/path: "/metrics"
```

When the exporter is running, nginx pods will show `2/2` containers.

---

### Part 5 — Grafana Tempo

> ⚠️ **Use chart version `1.23.3`.** Version `1.24.x` contains a deprecated `mem-ballast` argument that causes `CrashLoopBackOff`.

Add the following to `apps/base/tempo/release.yaml`:

```yaml
spec:
  chart:
    spec:
      chart: tempo
      version: "1.23.3"   # Do NOT use 1.24.x
  values:
    tempo:
      storage:
        trace:
          backend: local
          local:
            path: /var/tempo/traces
          wal:
            path: /var/tempo/wal
    persistence:
      enabled: true
      size: 10Gi
```

**Verify:**

```bash
kubectl get pods -n tempo
curl -s -o /dev/null -w "%{http_code}" https://YOUR_DOMAIN/tempo/ready  # Expect: 200
```

Tempo listens on port `3200` for HTTP/UI and port `4318` for OTLP HTTP push.

---

### Part 6–7 — Loki & FluentBit

> ⚠️ **Place the Fluent `HelmRepository` in `infrastructure/configs/`**, not in `apps/base/fluent-bit/`. Placing it in base causes it to be included in both staging and production, resulting in a duplicate resource error.

**Loki** — deploy in SingleBinary mode (`apps/base/loki/release.yaml`):

> ⚠️ Pin `sidecar.image.tag: 1.28.0`. Versions `2.x` have an SSL certificate bug causing `CrashLoopBackOff`.

```yaml
values:
  deploymentMode: SingleBinary
  loki:
    auth_enabled: false
    storage:
      type: filesystem
  singleBinary:
    replicas: 1
  read:
    replicas: 0
  write:
    replicas: 0
  backend:
    replicas: 0
  gateway:
    enabled: false
  chunksCache:
    enabled: false
  resultsCache:
    enabled: false
  sidecar:
    image:
      tag: "1.28.0"   # Pin this version
    rules:
      enabled: false
```

**FluentBit** — configure output to point directly at Loki (not the gateway):

```ini
[OUTPUT]
    Name   loki
    Match  kube.*
    Host   loki.loki.svc.cluster.local
    Port   3100
    Labels job=fluentbit
    Auto_kubernetes_labels on
```

**Verify logs are flowing:**

```bash
kubectl get pods -n loki        # loki-0 should be 1/1 Running
kubectl get pods -n fluent-bit  # One pod per node

kubectl logs -l app.kubernetes.io/name=fluent-bit -n fluent-bit --tail=3 | grep hostname
# Expected: hostname=loki.loki.svc.cluster.local:3100
```

---

### Part 8 — External Access

Each service exposes a secured HTTPS push endpoint protected by BasicAuth middleware:

| Service | Path | Backend |
|---------|------|---------|
| Prometheus Pushgateway | `/push` | `prometheus-prometheus-pushgateway:9091` |
| Loki | `/loki/api/v1/push` | `loki:3100` |
| Tempo | `/v1/traces` | `tempo:4318` |

> ℹ️ Prometheus also requires a strip-prefix middleware since Pushgateway serves on root.

**Test that unauthenticated requests are rejected:**

```bash
curl -s -o /dev/null -w "push: %{http_code}\n" https://YOUR_DOMAIN/push
curl -s -o /dev/null -w "loki: %{http_code}\n" https://YOUR_DOMAIN/loki/api/v1/push
curl -s -o /dev/null -w "tempo: %{http_code}\n" https://YOUR_DOMAIN/v1/traces
# All should return 401
```

---

### Part 9 — Grafana Dashboards

**Add datasources to `apps/base/grafana/release.yaml`:**

```yaml
values:
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          uid: prometheus
          url: http://prometheus-server.prometheus.svc.cluster.local
          access: proxy
          isDefault: true
        - name: Loki
          type: loki
          uid: loki
          url: http://loki.loki.svc.cluster.local:3100
          access: proxy
        - name: Tempo
          type: tempo
          uid: tempo
          url: http://tempo.tempo.svc.cluster.local:3200
          access: proxy
```

**Auto-provision dashboards** by placing ConfigMaps with the label `grafana_dashboard: "1"` in `apps/base/grafana/dashboards/`. Grafana's sidecar picks these up automatically — no UI interaction required.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-metrics-dashboard
  namespace: grafana
  labels:
    grafana_dashboard: "1"
data:
  nginx-metrics.json: |
    { ... dashboard JSON ... }
```

---

## Verification

Run through this checklist after full deployment:

| # | Component | Expected | Command |
|---|-----------|----------|---------|
| 1 | Prometheus (no auth) | `401` | `curl /prometheus` |
| 2 | Prometheus (with auth) | `302` | `curl -u admin:pass /prometheus` |
| 3 | Grafana | `302` | `curl /grafana` |
| 4 | Nginx | `200` | `curl /nginx` |
| 5 | Tempo ready | `200` | `curl /tempo/ready` |
| 6 | Nginx metrics | `2/2` pods | `kubectl get pods -n nginx` |
| 7 | Loki running | `1/1 Running` | `kubectl get pods -n loki` |
| 8 | FluentBit running | 1 per node | `kubectl get pods -n fluent-bit` |
| 9 | Push (no auth) | `401` | `curl /push` |
| 10 | Push (with auth) | `200` | `curl -u external:pass /push` |
| 11 | Loki push (no auth) | `401` | `curl /loki/api/v1/push` |
| 12 | Loki push (with auth) | `204` | `curl -u external:pass -X POST /loki/api/v1/push` |
| 13 | Tempo push (no auth) | `401` | `curl /v1/traces` |
| 14 | Tempo push (with auth) | `200` | `curl -u external:pass -X POST /v1/traces` |
| 15 | Grafana datasources | 3 sources | `curl /grafana/api/datasources` |
| 16 | Grafana dashboards | 3+ dashboards | `curl /grafana/api/search` |

---

## External Push Endpoints

External systems (CI pipelines, applications, other clusters) can push observability data over authenticated HTTPS:

```bash
# Push metrics
curl -u external:YOUR_EXT_PASSWORD https://YOUR_DOMAIN/push

# Push logs
curl -u external:YOUR_EXT_PASSWORD \
  -X POST -H 'Content-Type: application/json' \
  -d '{"streams":[{"stream":{"job":"external","namespace":"external"},"values":[["'"$(date +%s)"'000000000","test"]]}]}' \
  https://YOUR_DOMAIN/loki/api/v1/push

# Push traces
curl -u external:YOUR_EXT_PASSWORD \
  -X POST -H 'Content-Type: application/json' \
  -d '{}' https://YOUR_DOMAIN/v1/traces
```

---

## Known Issues

| Issue | Solution |
|-------|----------|
| Tempo `CrashLoopBackOff` | Use chart version `1.23.3`, not `1.24.x` |
| Loki sidecar SSL error / crash | Pin `sidecar.image.tag: 1.28.0` |
| Loki `502 Bad Gateway` | Disable gateway; use `SingleBinary` mode |
| FluentBit `domain not found` | Use `loki.loki.svc.cluster.local:3100` directly |
| Duplicate `HelmRelease` error | Place fluent repo in `infrastructure/configs/`, not `apps/base/fluent-bit/` |
| `ServiceMonitor` not found in staging | Use pod annotations instead of `ServiceMonitor` CRD |
| Grafana datasources not loading | Run: `flux reconcile helmrelease grafana -n grafana --reset` |

---

## Contributing

Contributions, issues, and pull requests are welcome. Please open an issue before submitting a large change so we can discuss the approach first.

1. Fork the repository
2. Create a feature branch (`git checkout -b feat/my-change`)
3. Commit your changes (`git commit -m 'feat: describe change'`)
4. Push the branch (`git push origin feat/my-change`)
5. Open a Pull Request

---

