# infra-monitoring
![infra_monitoring_stack_arrows_v3](https://github.com/user-attachments/assets/884d2abe-5a3f-4bc5-9831-61cc62827e42)

Full-stack Kubernetes observability across three layers — OS, Kubernetes, and application. Metrics and logs collected by OpenTelemetry Collector, stored in VictoriaMetrics and VictoriaLogs, visualised in Grafana. One `kubectl apply` bootstraps everything via ArgoCD.

## Stack

| Component | Role |
|---|---|
| OpenTelemetry Collector | DaemonSet on every node — collects all signals |
| VictoriaMetrics | Metrics storage — 1-month retention |
| VictoriaLogs | Log storage — 30-day retention |
| Grafana | Unified UI — datasources and dashboards auto-provisioned |
| Envoy Gateway | Gateway API ingress — HTTP + HTTPS with HTTP/2 |
| cert-manager | Automatic TLS certificate provisioning |

## Telemetry layers

Each layer has dedicated receivers, its own processor chain, and a `layer` label for clean routing in Grafana.

**OS — `layer=infra`**
Host-level collection via direct mounts (`/proc`, `/sys`, `/run/log/journal`). Receivers: `hostmetrics` (CPU, memory, disk, network, processes), `journald` (systemd logs), `filelog/system`.

**Kubernetes — `layer=kuber`**
Workload signals enriched with full k8s metadata. Receivers: `kubeletstats` (pod CPU/memory/network), `k8s_events` (cluster events). Processor: `k8sattributes` adds namespace, pod, deployment, and node labels to every signal.

**Application — `layer=app`**
Annotation-driven, zero-config discovery. Any pod with `prometheus.io/scrape: "true"` is scraped automatically. Receivers: `prometheus/app` (request rate, latency, custom metrics), `filelog/app` (CRI-parsed container logs with pod/namespace attribution).

## Pipelines

```
hostmetrics + kubeletstats  →  resourcedetection · k8sattributes · batch  →  VictoriaMetrics
prometheus/app              →  resource/app · k8sattributes · batch       →  VictoriaMetrics

journald + filelog/system   →  resource/infra · k8sattributes · batch     →  VictoriaLogs
k8s_events                  →  resource/kuber · batch                     →  VictoriaLogs
filelog/app                 →  resource/app · k8sattributes · batch       →  VictoriaLogs
```

## Dashboards

| Dashboard | Panels |
|---|---|
| Node Infrastructure | CPU, memory, disk, network per node · system logs |
| Kubernetes Infrastructure | Pod CPU/memory/network · k8s events log |
| App — agents-infra-test | Request rate, p95 latency · application logs |

All panels use the `layer` label as the primary filter (`{layer="infra"}`, `{layer="kuber"}`, `{layer="app"}`).

## Routing

All services exposed via Envoy Gateway using the Kubernetes Gateway API. TLS terminates at the Gateway level — cert-manager provisions a self-signed wildcard cert for `*.local` automatically.

| Service | URL | Protocol |
|---|---|---|
| Grafana | `https://grafana.local` | HTTPS |
| VictoriaMetrics | `http://victoriametrics.local` | HTTP |
| VictoriaLogs | `http://victorialogs.local` | HTTP |
| ArgoCD | `http://argocd.local` | HTTP |

## Bootstrap

```bash
kubectl apply -f argocd/app-of-apps.yaml
```

Add to `/etc/hosts`:

```
<node-ip> grafana.local victoriametrics.local victorialogs.local argocd.local
```

Then open `https://grafana.local`. Anonymous admin access is enabled by default.

## Design notes

**Static otelcol config** — no Helm `{{ }}` inside the block scalar. Component addresses are hardcoded Kubernetes service DNS names.

**ConfigMap-provisioned dashboards** — datasources and dashboards survive pod restarts and cluster rebuilds without re-importing.

**VictoriaMetrics + VictoriaLogs over Prometheus + Loki** — single-binary each, lower memory footprint, native OpenTelemetry ingestion on VictoriaLogs.

**Gateway API** — uses `HTTPRoute` resources instead of `Ingress`. TLS terminates at the shared `homelab-gateway` in `envoy-gateway-system`. Cross-namespace routing permitted via `ReferenceGrant` resources.

**cert-manager** — self-signed `ClusterIssuer` provisions `homelab-tls` secret in `envoy-gateway-system`, referenced directly by the Gateway HTTPS listener.