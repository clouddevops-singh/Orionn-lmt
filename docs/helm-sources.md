# Upstream Helm sources

Repository URLs and chart names checked against upstream documentation on 2026-10-19. Pin tested versions in deployment configuration before use; do not install floating latest versions in production.

| Component | Helm repository | Chart | Upstream documentation |
|---|---|---|---|
| Mimir | `https://grafana.github.io/helm-charts` (`grafana`) | `mimir-distributed` | [Mimir Helm install](https://grafana.com/docs/helm-charts/mimir-distributed/latest/get-started-helm-charts/) |
| Tempo | `https://grafana.github.io/helm-charts` (`grafana`) | `tempo-distributed` or `tempo` | [Tempo Helm deployment](https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/deploy/kubernetes/helm-chart/) |
| Loki | `https://grafana-community.github.io/helm-charts` (`grafana-community`) | `loki` | [Loki Helm install](https://grafana.com/docs/loki/latest/setup/install/helm/install-scalable/) |
| Grafana | `https://grafana-community.github.io/helm-charts` (`grafana-community`) | `grafana` | [Grafana Helm install](https://grafana.com/docs/grafana/latest/setup-grafana/installation/helm/) |
| Fluent Bit / Fluentd | `https://fluent.github.io/helm-charts` (`fluent`) | `fluent-bit`, `fluentd` | [Fluent Helm charts](https://github.com/fluent/helm-charts) |
| OpenTelemetry Collector | `https://open-telemetry.github.io/opentelemetry-helm-charts` (`open-telemetry`) | `opentelemetry-collector` | [Collector Helm chart](https://opentelemetry.io/docs/platforms/kubernetes/helm/collector/) |

Notes:

- Grafana publishes Mimir and Tempo charts from its Helm repository.
- The Loki Helm chart moved to the Grafana Community chart repository in March 2026; use the current repo and review its migration notes.
- The Fluent chart repository documents both OCI installation and the `fluent` Helm repository; this project uses the latter for straightforward Helm examples.
- OpenTelemetry Collector chart mode must be selected explicitly (`daemonset`, `deployment`, or `statefulset`) to match the source-cluster pipeline design.
- Recent Mimir ingest-storage and Tempo 3 releases have Kafka-compatible queue requirements. Check release-specific documentation before choosing versions or deploying a shared queue.
