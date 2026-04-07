# infra-monitoring

GitOps-managed observability stack for the homelab k3s cluster.

## Stack

| Component | Kind | Purpose |
|---|---|---|
| otelcol | DaemonSet | Collect host + k8s metrics and logs from every node |
| VictoriaMetrics | Deployment + PVC | Store metrics, 1 month retention |
| VictoriaLogs | Deployment + PVC | Store logs, 30 day retention |
| Grafana | Deployment + PVC | Dashboards at grafana.local |

## What otelcol collects

**Per node — host level (via host mounts):**
- CPU, memory, disk, network, processes — `hostmetrics` receiver via `/hostfs`
- Systemd journal — `journald` receiver via `/run/log/journal`
- Log files — `filelog` receiver via `/var/log`

**Kubernetes workloads:**
- Pod/container/node resource metrics — `kubeletstats` receiver
- Cluster events — `k8s_events` receiver
- Metadata enrichment on every metric and log line — namespace, pod, deployment, node


## Bootstrap

One command — ArgoCD does everything from there:

```bash
kubectl apply -f argocd/app-of-apps.yaml
```

ArgoCD creates the `monitoring` namespace and deploys all four components.

## Access Grafana

Add to `/etc/hosts`:
```
<kserver-ip>  grafana.local
```

Open `https://grafana.local`. VictoriaMetrics and VictoriaLogs datasources
are provisioned automatically on first start.

## Storage layout

VictoriaMetrics and VictoriaLogs are pinned to `kserver` via `nodeSelector`
(212GB SSD). PVCs use the `local-path` provisioner bundled with k3s.

Retention is enforced by the binaries — no cron jobs needed:
- Metrics: 1 month (`--retentionPeriod=1`)
- Logs: 30 days (`--retentionPeriod=30d`)

## Verify node label before deploying

```bash
kubectl get nodes --show-labels | grep hostname
```

The `nodeSelector` in `victoriametrics/values.yaml` and `victorialogs/values.yaml`
expects `kubernetes.io/hostname: kserver`. Update if your node name differs.