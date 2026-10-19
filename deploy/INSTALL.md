# Pinned versions, installation, and verification

Version matrix and command templates for the private product-cluster / source-cluster design. Upstream versions were checked on **2026-10-19**. Recheck upstream releases and the Kubernetes compatibility notes before production upgrades. Chart versions are pinned; do not replace them with floating `latest` tags.

## Version matrix

| Component | Cluster | Helm repo / chart | Chart version | App version |
|---|---|---|---:|---:|
| Grafana Mimir | Product | `grafana/mimir-distributed` | `6.2.1` | `3.2.1` |
| Grafana Loki | Product | `grafana-community/loki` | `18.17.1` | `3.7.8` |
| Grafana Tempo, distributed | Product | `grafana-community/tempo-distributed` | `3.14.1` | `3.1.0` |
| Grafana | Product | `grafana-community/grafana` | `14.0.1` | `13.2.4` |
| Prometheus Operator stack | Source | `prometheus-community/kube-prometheus-stack` | `92.10.1` | Operator `v0.94.1` |
| Prometheus server | Source | pinned in the stack values | n/a | `v3.15.0` |
| Fluent Bit | Source | `fluent/fluent-bit` | `0.58.3` | `5.1.3` |
| Fluentd | Source | `fluent/fluentd` | `0.6.0` | `1.19.3` |
| OpenTelemetry Collector | Source | `open-telemetry/opentelemetry-collector` | `0.176.0` | `0.162.0` |

The kube-prometheus-stack chart's `appVersion` is the Prometheus Operator version, not the Prometheus server version. Set the server image tag to `v3.15.0` in its values file. The source-cluster values should disable the chart's bundled Grafana if you use the central Grafana release.

## Before installing

Run these commands from the repository root in Bash, WSL, or Git Bash. Replace the context names with the entries from `kubectl config get-contexts`. Helm's `--kube-context` flag keeps every release targeted at the intended private cluster.

```bash
kubectl config get-contexts
export PRODUCT_CONTEXT="private-product-cluster"
export SOURCE_CONTEXT="private-source-cluster-a"
export OBS_NS="observability"
export LOG_NS="logging"
export METRICS_NS="monitoring"
export OTEL_NS="opentelemetry"

helm repo add grafana https://grafana.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo add fluent https://fluent.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

Create a values file for each release before installing. Start with the defaults for the **pinned** chart version, then set your private object-store buckets, Kafka-compatible queue, storage classes, TLS/authentication, source-cluster identity, and resource sizing:

```bash
mkdir -p deploy/product-cluster/values deploy/source-cluster/values
helm show values grafana/mimir-distributed --version 6.2.1 > deploy/product-cluster/values/mimir.yaml
helm show values grafana-community/loki --version 18.17.1 > deploy/product-cluster/values/loki.yaml
helm show values grafana-community/tempo-distributed --version 3.14.1 > deploy/product-cluster/values/tempo.yaml
helm show values grafana-community/grafana --version 14.0.1 > deploy/product-cluster/values/grafana.yaml
helm show values prometheus-community/kube-prometheus-stack --version 92.10.1 > deploy/source-cluster/values/kube-prometheus-stack.yaml
helm show values fluent/fluent-bit --version 0.58.3 > deploy/source-cluster/values/fluent-bit.yaml
helm show values fluent/fluentd --version 0.6.0 > deploy/source-cluster/values/fluentd.yaml
helm show values open-telemetry/opentelemetry-collector --version 0.176.0 > deploy/source-cluster/values/otel-collector.yaml
```

Keep secret values out of these files and out of Git. Reference Kubernetes Secrets or an external-secrets/workload-identity integration. Recent Mimir ingest storage and Tempo 3 use a Kafka-compatible queue; configure the queue required by these pinned chart versions rather than relying on a single-node demo dependency for production. Use external durable object storage for production data.

## Install on the product cluster

Review and render each release before applying it. These commands require the corresponding values file prepared above.

```bash
helm upgrade --install mimir grafana/mimir-distributed \
  --version 6.2.1 --namespace "$OBS_NS" --create-namespace \
  --kube-context "$PRODUCT_CONTEXT" \
  --values deploy/product-cluster/values/mimir.yaml

helm upgrade --install loki grafana-community/loki \
  --version 18.17.1 --namespace "$OBS_NS" --create-namespace \
  --kube-context "$PRODUCT_CONTEXT" \
  --values deploy/product-cluster/values/loki.yaml

helm upgrade --install tempo grafana-community/tempo-distributed \
  --version 3.14.1 --namespace "$OBS_NS" --create-namespace \
  --kube-context "$PRODUCT_CONTEXT" \
  --values deploy/product-cluster/values/tempo.yaml

helm upgrade --install grafana grafana-community/grafana \
  --version 14.0.1 --namespace "$OBS_NS" --create-namespace \
  --kube-context "$PRODUCT_CONTEXT" \
  --values deploy/product-cluster/values/grafana.yaml
```

Use the pinned chart's production configuration for storage, Kafka, authentication, and replicas. Do not expose the backend services, Kafka, or object storage through the cross-cluster ingress; expose only the authenticated telemetry write APIs and a separate SSO-protected Grafana route.

## Install on each source cluster

Repeat for every source cluster, with a per-cluster Prometheus values file and stable `cluster` label/resource attribute. Fluent Bit forwards logs to the Fluentd aggregator; Prometheus remote-writes metrics to Mimir; the OpenTelemetry Collector exports OTLP traces to Tempo.

```bash
helm upgrade --install fluent-bit fluent/fluent-bit \
  --version 0.58.3 --namespace "$LOG_NS" --create-namespace \
  --kube-context "$SOURCE_CONTEXT" \
  --values deploy/source-cluster/values/fluent-bit.yaml

helm upgrade --install fluentd fluent/fluentd \
  --version 0.6.0 --namespace "$LOG_NS" --create-namespace \
  --kube-context "$SOURCE_CONTEXT" \
  --values deploy/source-cluster/values/fluentd.yaml

helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --version 92.10.1 --namespace "$METRICS_NS" --create-namespace \
  --kube-context "$SOURCE_CONTEXT" \
  --values deploy/source-cluster/values/kube-prometheus-stack.yaml

helm upgrade --install otel-collector open-telemetry/opentelemetry-collector \
  --version 0.176.0 --namespace "$OTEL_NS" --create-namespace \
  --kube-context "$SOURCE_CONTEXT" \
  --values deploy/source-cluster/values/otel-collector.yaml
```

In the kube-prometheus-stack values, pin the Prometheus server image to `quay.io/prometheus/prometheus:v3.15.0`, set `prometheus.prometheusSpec.externalLabels.cluster` to this source cluster's ID, and configure `prometheus.prometheusSpec.remoteWrite` to the private Mimir write URL. Configure Secret references and TLS trust for the gateway. In the Fluent values, configure the Forward output/input between Fluent Bit and Fluentd and Fluentd's authenticated Loki output. In the Collector values, choose a supported mode and configure OTLP export to the Tempo gateway.

## Verify Kubernetes and Helm releases

Run these after each installation. All releases should report `deployed`; workloads should become Ready; PVCs should bind; and recent Events should not show failed scheduling, image pulls, mounts, or probes.

```bash
helm list --all-namespaces --kube-context "$PRODUCT_CONTEXT"
kubectl --context "$PRODUCT_CONTEXT" -n "$OBS_NS" get pods,svc,ingress,pvc -o wide
kubectl --context "$PRODUCT_CONTEXT" -n "$OBS_NS" get events --sort-by=.lastTimestamp

helm list --all-namespaces --kube-context "$SOURCE_CONTEXT"
kubectl --context "$SOURCE_CONTEXT" -n "$LOG_NS" get daemonsets,deployments,pods,svc -o wide
kubectl --context "$SOURCE_CONTEXT" -n "$METRICS_NS" get prometheus,pods,svc -o wide
kubectl --context "$SOURCE_CONTEXT" -n "$OTEL_NS" get daemonsets,deployments,pods,svc -o wide
kubectl --context "$SOURCE_CONTEXT" -A get events --sort-by=.lastTimestamp
```

Check release-specific status and logs when a pod is not Ready:

```bash
helm status mimir -n "$OBS_NS" --kube-context "$PRODUCT_CONTEXT"
helm status loki -n "$OBS_NS" --kube-context "$PRODUCT_CONTEXT"
helm status tempo -n "$OBS_NS" --kube-context "$PRODUCT_CONTEXT"
helm status grafana -n "$OBS_NS" --kube-context "$PRODUCT_CONTEXT"
kubectl --context "$PRODUCT_CONTEXT" -n "$OBS_NS" describe pod <pod-name>
kubectl --context "$PRODUCT_CONTEXT" -n "$OBS_NS" logs <pod-name> --all-containers --tail=100
```

## Verify private endpoints and incoming telemetry

Set each URL to the authenticated internal ingress/gateway route reachable from the relevant source cluster. Supply bearer credentials from a secret manager; do not paste tokens into Git or shell history. These readiness endpoints should return HTTP 200:

```bash
curl -fsS "$MIMIR_WRITE_URL/ready"
curl -fsS "$LOKI_URL/ready"
curl -fsS "$TEMPO_URL/ready"
curl -fsS "$GRAFANA_URL/api/health"
```

After the source collectors are running, confirm recent samples/logs/traces in Grafana Explore using the source cluster's `cluster` identity. If the gateway maps Mimir's query API under `/prometheus`, query it using that configured prefix:

```bash
# Mimir: MIMIR_QUERY_URL includes the configured Prometheus query prefix, if any.
curl -GfsS "$MIMIR_QUERY_URL/api/v1/query" \
  --data-urlencode 'query=up{cluster="source-cluster-a"}'

# Loki: use the gateway's tenant identity or its approved tenant header.
curl -GfsS "$LOKI_URL/loki/api/v1/query_range" \
  --data-urlencode 'query={cluster="source-cluster-a"}' \
  --data-urlencode 'limit=5'
```

Expected results: Mimir's Prometheus API returns a `success` response with `up` series after scraping starts; Loki returns a `success` response with recent log entries after Fluentd forwards logs; Grafana's Explore view can query Tempo and show traces after an instrumented application exports OTLP. An empty result usually means routing, credentials/tenant mapping, labels, collector configuration, or simply no data has arrived yet; check the corresponding source collector logs and gateway access logs.

## Diagnose source pipelines

```bash
kubectl --context "$SOURCE_CONTEXT" -n "$LOG_NS" logs daemonset/fluent-bit --all-containers --tail=100
kubectl --context "$SOURCE_CONTEXT" -n "$LOG_NS" logs deployment/fluentd --all-containers --tail=100
kubectl --context "$SOURCE_CONTEXT" -n "$METRICS_NS" get pods
kubectl --context "$SOURCE_CONTEXT" -n "$OTEL_NS" get pods
kubectl --context "$SOURCE_CONTEXT" -n "$OTEL_NS" logs deployment/otel-collector --all-containers --tail=100
```

Fluent Bit and Fluentd logs should show successful forwarding without retry/buffer growth; Prometheus should have active scrape targets and a healthy remote-write queue; the Collector should show successful span export without persistent send failures. Verify representative log fields, metric labels, and trace resource attributes all carry the same `cluster` value so dashboards and cross-signal links work.

## Upstream version references

- [Mimir 6.2 chart line](https://github.com/grafana/mimir/tree/mimir-distributed-release-6.2/operations/helm/charts/mimir-distributed)
- [Loki chart metadata](https://github.com/grafana-community/helm-charts/blob/main/charts/loki/Chart.yaml)
- [Tempo distributed chart metadata](https://github.com/grafana-community/helm-charts/blob/main/charts/tempo-distributed/Chart.yaml)
- [Grafana chart metadata](https://github.com/grafana-community/helm-charts/blob/main/charts/grafana/Chart.yaml)
- [Fluent Bit chart metadata](https://github.com/fluent/helm-charts/blob/main/charts/fluent-bit/Chart.yaml)
- [Fluentd chart metadata](https://github.com/fluent/helm-charts/blob/main/charts/fluentd/Chart.yaml)
- [Prometheus chart metadata](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus/Chart.yaml)
- [Kubernetes Collector chart releases](https://github.com/open-telemetry/opentelemetry-helm-charts/releases)

