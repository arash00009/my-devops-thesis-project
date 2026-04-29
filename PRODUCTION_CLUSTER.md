PRODUCTION CLUSTER — lia-prod.cloudist.solutions
═══════════════════════════════════════════════════════════════════════════════════════════════

  INTERNET
      │ :80 (HTTP)   :443 (HTTPS)
      ▼
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│ ns-traefik                                                          [no NetworkPolicy]     │
│   pod-traefik  Deployment ×1                                                               │
│     :80  → 301 permanent redirect → HTTPS  (pass-through: /.well-known/acme-challenge/*)   │
│     :443 → TLS termination → path-based routing to cluster services                        │
│     :9182  windows-exporter metrics (cluster-internal only)                                │
└──┬──────────────┬──────────────────┬────────────────┬──────────────┬───────────────────────┘
   │ /nginx       │ /grafana         │ /prometheus    │ /loki        │ /tempo
   │ → svc:80     │ → svc:80         │ → svc:80       │ → svc:3100   │ → svc:3200
   │              │                  │ /alertmanager  │              │
   │              │                  │ → svc:9093     │              │
   ▼              │                  ▼                ▼              ▼
┌──────────────┐  │   ┌──────────────────────┐ ┌──────────────┐ ┌──────────────┐
│  ns-nginx    │  │   │    ns-prometheus     │ │   ns-loki    │ │   ns-tempo   │
│              │  │   │                      │ │              │ │              │
│  pod-nginx   │  │   │  pod-prometheus-     │ │  pod-loki    │ │  pod-tempo   │
│  Dep. ×3     │  │   │    server            │ │  SS ×1       │ │  SS ×1       │
│  :80         │  │   │    SS ×1  :9090      │ │  :3100       │ │  :3200       │
│  :9113 ◄─────┼──┼───┤    8 GiB PVC         │ │  TSDB v13    │ │  :4317 (recv)│
│   metrics    │  │   │                      │ │  10 GiB      │ │  WAL 10 GiB  │
└──────────────┘  │   │  pod-prometheus-     │ └──────┬───────┘ └──────┬───────┘
                  │   │    alertmanager      │        │                │
                  │   │    SS ×1  :9093      │        │ :3100          │ :4317
                  │   │                      │        │ (log ingest)   │ (trace ingest)
                  │   │  pod-kube-state-     │   ┌────▼──────────┐ ┌──▼────────────┐
                  │   │    metrics  Dep. ×1  │   │ ns-fluent-bit │ │   ns-alloy    │
                  │   │    :8080             │   │               │ │               │
                  │   └──────────────────────┘   │ pod-fluent-   │ │  pod-alloy    │
                  │                              │   bit         │ │  DaemonSet    │
                  │                              │  DaemonSet    │ │  (per node)   │
                  │                              │  (per node)   │ │  :4317 OTLP   │
                  │                              │  tails        │ │  :4318 OTLP   │
                  │                              │  /var/log/    │ │  gRPC/HTTP    │
                  │                              │  containers/  │ │  recv         │
                  │                              │  nginx*.log   │ └───────────────┘
                  │                              └───────────────┘
                  ▼
      ┌───────────────────────────────────────────────────────────────────────────────┐
      │ ns-grafana                                                [no NetworkPolicy]  │
      │   pod-grafana  Dep. ×1  :3000 → svc:80                                       │
      │   pod-grafana-sidecar  (watches ConfigMaps label grafana_dashboard=1)         │
      │                                                                               │
      │   datasources:                                                                │
      │     prometheus-server.prometheus.svc.cluster.local:80  (metrics)             │
      │     loki.loki.svc.cluster.local:3100                   (logs)                │
      │     tempo.tempo.svc.cluster.local:3200                 (traces)              │
      │                                                                               │
      │   dashboards: nginx-metrics · nginx-logs · traces                            │
      └───────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────────────────────┐
│ INFRASTRUCTURE                                                                            │
│                                                                                           │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ ns-cert-manager                                               [no NetworkPolicy]    │  │
│  │   pod-cert-manager            Dep. ×1   :9402 metrics                              │  │
│  │   pod-cert-manager-cainjector Dep. ×1                                              │  │
│  │   pod-cert-manager-webhook    Dep. ×1   :6060 webhook  :9403 metrics               │  │
│  │                                                                                    │  │
│  │   ClusterIssuer: letsencrypt (HTTP-01 ACME via acme-v02.api.letsencrypt.org)       │  │
│  │   issues TLS certs for all Ingress resources across the cluster                    │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                           │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ ns-kube-system                                                [no NetworkPolicy]    │  │
│  │   pod-local-path-provisioner  Dep. ×1                                              │  │
│  │   default StorageClass → backs PVCs: prometheus 8 GiB · loki 10 GiB · tempo 10 GiB│  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                           │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ ns-flux-system                                         [3 NetworkPolicies — see ▼] │  │
│  │   pod-source-controller        Dep. ×1   :9090 API  :8080 metrics  :9440 health    │  │
│  │   pod-kustomize-controller     Dep. ×1              :8080 metrics  :9440 health    │  │
│  │   pod-helm-controller          Dep. ×1              :8080 metrics  :9440 health    │  │
│  │   pod-notification-controller  Dep. ×1   :9090/:9292 webhooks  :8080 metrics       │  │
│  │                                                                                    │  │
│  │   reconciles devops-fluxcd Git repo → desired state for all namespaces             │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════════════════════
FIREWALL RULES (Kubernetes NetworkPolicy)
═══════════════════════════════════════════════════════════════════════════════════════════════

  ns-flux-system  — only namespace with NetworkPolicy enforced

  ┌─────────────────────────────────────────────────────────────────────────────────────┐
  │ Policy: allow-egress                                                                │
  │   applies to : all pods in ns-flux-system                                          │
  │   INGRESS     allow from: pods within ns-flux-system only (intra-namespace)        │
  │   EGRESS      allow all outbound (unrestricted)                                    │
  └─────────────────────────────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────────────────────────────┐
  │ Policy: allow-scraping                                                              │
  │   applies to : all pods in ns-flux-system                                          │
  │   INGRESS     allow from: ANY namespace  →  :8080 TCP (Prometheus metrics)         │
  └─────────────────────────────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────────────────────────────┐
  │ Policy: allow-webhooks                                                              │
  │   applies to : pod-notification-controller                                         │
  │   INGRESS     allow from: ANY namespace  →  any port (webhook delivery)            │
  └─────────────────────────────────────────────────────────────────────────────────────┘

  All other namespaces: no NetworkPolicy → Kubernetes default (all ingress/egress open)
    ns-cert-manager  ns-traefik  ns-kube-system  ns-nginx  ns-prometheus
    ns-loki  ns-tempo  ns-grafana  ns-fluent-bit  ns-alloy

═══════════════════════════════════════════════════════════════════════════════════════════════
INTER-SERVICE CONNECTIONS SUMMARY
═══════════════════════════════════════════════════════════════════════════════════════════════

  pod-traefik           → pod-nginx :80                    HTTP proxy
  pod-traefik           → pod-grafana :80                  HTTP proxy
  pod-traefik           → pod-prometheus-server :80        HTTP proxy  (BasicAuth middleware)
  pod-traefik           → pod-prometheus-alertmanager :9093 HTTP proxy
  pod-traefik           → pod-loki :3100                   HTTP proxy
  pod-traefik           → pod-tempo :3200                  HTTP proxy
  pod-prometheus-server → pod-nginx :9113                  Prometheus scrape (metrics)
  pod-grafana           → pod-prometheus-server :80        datasource query
  pod-grafana           → pod-loki :3100                   datasource query
  pod-grafana           → pod-tempo :3200                  datasource query
  pod-fluent-bit        → pod-loki :3100                   log shipping (HTTP)
  pod-alloy             → pod-tempo :4317                  trace forwarding (OTLP gRPC)
 