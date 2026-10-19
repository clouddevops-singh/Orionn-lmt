# Product-cluster deployment plan

Install these releases only after completing phase 0 and phase 1 in the roadmap. The commands below register upstream repositories and show chart names; they intentionally omit chart versions and credentials until the Kubernetes, chart, storage, and queue compatibility decisions are recorded.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update

# Inspect available versions and upstream values before locking versions.
helm search repo grafana/mimir-distributed --versions
helm search repo grafana/tempo-distributed --versions
helm search repo grafana-community/loki --versions
helm search repo grafana-community/grafana --versions
```

Planned release order:

1. Private storage and a Kafka-compatible queue, when required by the selected chart/app versions.
2. Internal ingress/authentication gateway and private DNS/TLS.
3. Mimir, Loki, and Tempo with private object storage and persistent volumes.
4. Grafana with SSO, provisioned data sources, dashboards, and alerts.

Expose only authenticated ingest endpoints to source-cluster networks. Keep backend, queue, and object-store service ports internal. Commit chart versions and rendered values only after reviewing secrets handling; use external secret references instead of literal passwords or tokens.
