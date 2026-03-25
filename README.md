# devops-fluxcd

# Deployment Guide – From Scratch

---

## Part 1 – Create the Kubernetes Cluster (Virtuozzo)

### Step 1 – Set up the Network

Log in to Virtuozzo and create a network with the following settings:

| Setting | Value |
|---------|-------|
| Name | c-net2 |
| IP Range (CIDR) | 10.1.2.0/24 |
| Gateway | 10.1.2.1 |

### Step 2 – Create the Kubernetes Cluster

In Virtuozzo, create a new Kubernetes cluster with these settings:

| Setting | Value |
|---------|-------|
| Cluster name | My-LIA-k8s |
| Kubernetes version | v1.31.2 |
| Network | c-net2: 10.1.2.0/24 |
| High availability | Unchecked |
| Master node flavor | gp-c2-m8 (2 vCPUs, 8 GiB RAM) |
| Storage policy | All-Flash Premium |
| Disk size | 20 GB |
| Worker flavor | gp-c1-m8 (1 vCPU, 8 GiB RAM) |
| Number of worker nodes | 1 |

### Step 3 – Download and Configure kubectl Access

After the cluster is created, click on it and download the kubeconfig file. Place it in your `.kube` folder:

```
C:\Users\YourName\.kube\
```

Set the KUBECONFIG environment variable in your terminal:

```bash
export KUBECONFIG="/c/Users/YourName/.kube/My-LIA-k8s-25_02_2026_13_39.kubeconfig"
kubectl cluster-info
```

Verify it works:

```bash
kubectl get nodes
kubectl config current-context
```

### Step 4 – Make the Configuration Permanent

To avoid setting the environment variable every time you open a terminal, add it to your shell profile:

```bash
vi ~/.bashrc
```

Add this line and save:

```bash
export KUBECONFIG="/c/Users/YourName/.kube/My-LIA-k8s-25_02_2026_13_39.kubeconfig"
```

Then reload:

```bash
source ~/.bashrc
echo $KUBECONFIG
```

---

## Part 2 – Install Tools

### Step 5 – Install Flux CLI

```bash
winget install -e --id FluxCD.Flux -v 2.4.0
```

Verify:

```bash
flux check --pre
```

### Step 6 – Install Helm

```bash
winget install -e --id Helm.Helm
```

Add the Bitnami repository:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
```

---

## Part 3 – Bootstrap GitOps with Flux

### Step 7 – Bootstrap Flux on the Cluster

Replace `YOUR_GITHUB_TOKEN` with a GitHub personal access token that has `repo` permissions.

```bash
export GITHUB_TOKEN=YOUR_GITHUB_TOKEN
export GITHUB_USER=YOUR_GITHUB_USERNAME

flux bootstrap github \
  --components-extra=source-watcher \
  --owner=cloudist-se \
  --repository=devops-fluxcd \
  --branch=main \
  --personal \
  --path=clusters/staging \
  --token-auth
```

Wait until Flux is ready:

```bash
flux check
kubectl get pods -n flux-system
flux get kustomizations -n flux-system
```

---

## Part 4 – Clone the Repository

### Step 8 – Clone the Repository

```bash
git clone https://github.com/cloudist-se/devops-fluxcd.git
cd devops-fluxcd
```

---

## Part 5 – Create Secrets Manually

### Step 9 – Create Prometheus Basic Auth Secret

Replace `YOUR_PASSWORD` with your chosen password.

```bash
# Generate bcrypt hash
docker run --rm httpd:2 htpasswd -nbBC 12 admin YOUR_PASSWORD
```

Copy the hash from the output (looks like `admin:$2y$12$...`), then run:

```bash
kubectl create secret generic prometheus-basic-auth \
  --from-literal=web-config.yml='basic_auth_users:
  admin: PASTE_HASH_HERE' \
  -n prometheus
```

```bash
kubectl create secret generic prometheus-exporter-auth \
  --from-literal=username=admin \
  --from-literal=password=YOUR_PASSWORD \
  -n prometheus
```

### Step 10 – Create Grafana Admin Secret

Replace `YOUR_GRAFANA_PASSWORD` with your chosen password.

```bash
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=YOUR_GRAFANA_PASSWORD \
  -n grafana
```

---

## Part 6 – Verify the Deployment

### Step 11 – Verify That Everything is Running

```bash
# Check that all Flux kustomizations are ready
kubectl get kustomization -n flux-system

# Check pods in each namespace
kubectl get pods -n prometheus
kubectl get pods -n grafana
kubectl get pods -n nginx
```

All kustomizations should show `READY: True`.

### Step 12 – Verify That All Services Are Accessible

```bash
curl -s -o /dev/null -w "Prometheus: %{http_code}\n" https://lia-kube.cloudist.solutions/prometheus -u admin:YOUR_PASSWORD
curl -s -o /dev/null -w "Alertmanager: %{http_code}\n" https://lia-kube.cloudist.solutions/alertmanager
curl -s -o /dev/null -w "Grafana: %{http_code}\n" https://lia-kube.cloudist.solutions/grafana
curl -s -o /dev/null -w "Nginx: %{http_code}\n" https://lia-kube.cloudist.solutions/nginx
```

Expected responses: `200` or `302` for all services.

---

## Part 7 – Windows Exporter and Grafana Alloy

### Step 13 – Install Windows Exporter (on Windows Machine)

Open PowerShell as Administrator:

```powershell
# Download and install
Invoke-WebRequest -Uri "https://github.com/prometheus-community/windows_exporter/releases/download/v0.29.2/windows_exporter-0.29.2-amd64.msi" -OutFile "$env:USERPROFILE\Downloads\windows_exporter.msi"
Start-Process msiexec.exe -Wait -ArgumentList "/I $env:USERPROFILE\Downloads\windows_exporter.msi /quiet"

# Verify it is running
Get-Service windows_exporter
```

Verify in Git Bash:

```bash
curl http://localhost:9182/metrics | head -5
```

### Step 14 – Install and Configure Grafana Alloy (on Windows Machine)

Open PowerShell as Administrator:

```powershell
winget install GrafanaLabs.Alloy
```

Create the configuration file. Replace `YOUR_PROMETHEUS_PASSWORD`:

```powershell
$config = @'
logging {
  level = "info"
}

prometheus.scrape "windows" {
  targets = [{
    __address__ = "localhost:9182",
  }]
  forward_to = [prometheus.remote_write.prometheus_k8s.receiver]
  scrape_interval = "15s"
}

prometheus.remote_write "prometheus_k8s" {
  endpoint {
    url = "https://lia-kube.cloudist.solutions/prometheus/api/v1/write"
    basic_auth {
      username = "admin"
      password = "YOUR_PROMETHEUS_PASSWORD"
    }
  }
}
'@

[System.IO.File]::WriteAllText("$env:USERPROFILE\alloy-config.alloy", $config, [System.Text.Encoding]::ASCII)
```

Create a data directory and start Alloy:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\alloy-data" -Force
cd "$env:USERPROFILE"
& "C:\Program Files\GrafanaLabs\Alloy\alloy-windows-amd64.exe" run "$env:USERPROFILE\alloy-config.alloy" --storage.path="$env:USERPROFILE\alloy-data" --server.http.listen-addr=127.0.0.1:12346
```

Keep the PowerShell window open – Alloy must be running to send metrics.

### Step 15 – Create a Desktop Shortcut to Start Alloy Automatically

```powershell
$shortcut = (New-Object -ComObject WScript.Shell).CreateShortcut("$env:USERPROFILE\Desktop\Start Alloy.lnk")
$shortcut.TargetPath = "powershell.exe"
$shortcut.Arguments = "-NoExit -Command `"cd '$env:USERPROFILE'; & 'C:\Program Files\GrafanaLabs\Alloy\alloy-windows-amd64.exe' run '$env:USERPROFILE\alloy-config.alloy' --storage.path='$env:USERPROFILE\alloy-data' --server.http.listen-addr=127.0.0.1:12346`""
$shortcut.WorkingDirectory = "$env:USERPROFILE"
$shortcut.Save()
```

Double-click "Start Alloy" on the desktop each time you want to start sending metrics.

---

## Part 8 – Verify Metrics and Dashboards

### Step 16 – Verify Metrics in Prometheus

Open the browser and go to:

```
https://lia-kube.cloudist.solutions/prometheus
```

Log in with `admin` and your password. Search for:

```
windows_cpu_time_total
```

You should see data from your Windows machine.

### Step 17 – Verify Grafana Dashboard

Open the browser and go to:

```
https://lia-kube.cloudist.solutions/grafana
```

Go to **☰ → Dashboards** and click on **Windows Exporter**. You should see CPU and Memory gauges with live data.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Kustomization not ready | `flux reconcile kustomization apps -n flux-system --with-source` |
| Pod in CrashLoopBackOff | `kubectl logs <pod-name> -n <namespace>` |
| 404 on a service | `kubectl rollout restart deployment traefik -n traefik` |
| HelmRelease failed | `flux suspend helmrelease <n> -n <namespace> && flux resume helmrelease <n> -n <namespace>` |
| Secret missing after namespace deletion | Re-run Step 9 and Step 10 |
| Alloy sending 401 Unauthorized | Check that the correct password is in the alloy-config.alloy file |
| kubectl not connecting | Verify KUBECONFIG is set: `echo $KUBECONFIG` |
