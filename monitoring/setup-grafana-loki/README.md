# How to Setup Grafana + Loki for Logging On Kubernetes Cluster

## Architecture

```
 ┌──────────────── Node ────────────────┐
 │  Pod  Pod  Pod                       │
 │   │    │    │   stdout/stderr        │
 │   ▼    ▼    ▼                        │
 │  /var/log/pods/*/*.log               │
 │          │  tail                     │
 │          ▼                           │
 │   Grafana Alloy (DaemonSet)          │
 └──────────┬───────────────────────────┘
            │ push  /loki/api/v1/push
            ▼
     Loki (stores + indexes by labels)
            ▲
            │ LogQL query
     Grafana (UI, dashboards, Explore)
```

The logging stack has 3 components:
1. **Grafana Alloy** - the agent. Runs on every node, tails container log files, adds labels (`namespace`, `pod`, `container`, `app`) and pushes them to Loki. Alloy replaces Promtail, which is deprecated and has reached end of life.
2. **Loki** - the log store. Unlike Elasticsearch, it indexes only the **labels**, not the full log text. That makes it cheap to run.
3. **Grafana** - the UI. You query logs with **LogQL**.

| File | Purpose |
|---|---|
| `loki-values.yaml` | Loki in `SingleBinary` mode with filesystem storage (lab setup) |
| `alloy-values.yaml` | Alloy DaemonSet config that ships `/var/log/pods` to Loki |
| `grafana-values.yaml` | Grafana with Loki (and Prometheus) data sources provisioned |
| `log-generator.yaml` | Demo app that writes JSON logs |

## Prerequisites

```bash
brew install helm
kind create cluster --config ../../kind/kind-config.yaml

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace monitoring
```

## Step 1: Deploy Loki

```bash
helm install loki grafana/loki -n monitoring -f loki-values.yaml

kubectl get pods -n monitoring -l app.kubernetes.io/name=loki
# NAME     READY   STATUS    RESTARTS   AGE
# loki-0   2/2     Running   0          60s
```

Check that Loki is ready:

```bash
kubectl port-forward -n monitoring svc/loki 3100:3100
curl http://localhost:3100/ready
# ready
```

## Step 2: Deploy Grafana Alloy (log collector)

```bash
helm install alloy grafana/alloy -n monitoring -f alloy-values.yaml

kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy -o wide
# One alloy pod per node
```

Alloy has a debug UI that shows the component pipeline and its health:

```bash
kubectl port-forward -n monitoring ds/alloy 12345:12345
# open http://localhost:12345
```

## Step 3: Deploy Grafana

```bash
helm install grafana grafana/grafana -n monitoring -f grafana-values.yaml

kubectl port-forward -n monitoring svc/grafana 3000:80
```

Open `http://localhost:3000` and log in with `admin` / `admin`. The Loki data source is already configured under **Connections → Data sources**.

## Step 4: Generate logs and query them

```bash
kubectl apply -f log-generator.yaml
```

In Grafana, go to **Explore**, select the **Loki** data source, and try these LogQL queries:

```logql
# All logs from the demo app
{namespace="default", app="log-generator"}

# Only error lines (line filter)
{app="log-generator"} |= "error"

# Parse JSON and filter on a field
{app="log-generator"} | json | level="error"

# Error rate per pod (a metric query, can be used in dashboard panels)
sum by (pod) (rate({app="log-generator"} | json | level="error" [1m]))

# Logs from the whole cluster control plane
{namespace="kube-system"}
```

Use **Drilldown → Logs** to browse logs without writing queries.

## Troubleshooting

| Symptom | Check |
|---|---|
| No data in Grafana | `kubectl logs -n monitoring ds/alloy`, and check the Alloy UI for unhealthy components |
| Alloy `429` / `rate limited` | Raise `loki.limits_config.ingestion_rate_mb` |
| Loki pod `Pending` | No default StorageClass: `kubectl get sc` |
| `entry too far behind` | Old log files were re-read after a restart. This is harmless |

## Cleanup

```bash
kubectl delete -f log-generator.yaml
helm uninstall grafana alloy loki -n monitoring
kubectl delete pvc -n monitoring -l app.kubernetes.io/name=loki
```

## Going to production

- Use `deploymentMode: SimpleScalable` or `Distributed` with object storage (S3, GCS, Azure Blob) instead of filesystem.
- Enable `auth_enabled: true` for multi-tenancy, and put the gateway in front of Loki.
- Keep label cardinality low. Never use labels like `user_id` or `request_id`; filter on those with `| json` at query time instead.
- Store the Grafana admin password in a Secret (`admin.existingSecret`) and enable persistence.

## Reference:
1. https://grafana.com/docs/loki/latest/setup/install/helm/
2. https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/
3. https://grafana.com/docs/loki/latest/query/
