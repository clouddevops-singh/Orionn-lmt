# Source-cluster deployment plan

Install the same agent set in each private source cluster, with a values file per cluster for its identity, labels, routing, and local buffer capacity. Do not put credentials in these files; use the platform secret manager or workload identity.

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm search repo fluent/fluent-bit --versions
helm search repo fluent/fluentd --versions
helm search repo open-telemetry/opentelemetry-collector --versions
```

Required configuration contract:

- `cluster`, `environment`, and `region` are stable identifiers attached to logs, metric labels, and trace resource attributes.
- Fluent Bit tails container logs, persists offsets, enriches Kubernetes metadata, and forwards to the Fluentd aggregator over the cluster network.
- Fluentd applies approved parsing/redaction, uses a bounded persistent buffer, and pushes logs to the authenticated private Loki endpoint.
- Prometheus or Alloy scrapes cluster and application targets, limits high-cardinality labels, and remote-writes to Mimir with retry and queue limits.
- OpenTelemetry Collector receives OTLP locally, batches and retries, and exports OTLP to Tempo using TLS and the source identity.
- Network policies permit only the required internal collector traffic and egress to the product cluster's telemetry ingress.

Tune disk buffers against measured ingestion and the agreed maximum central-outage duration. Alert before buffers fill; define drop/backpressure behavior explicitly.
