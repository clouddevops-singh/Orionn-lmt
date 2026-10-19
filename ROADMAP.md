# Implementation roadmap

## Phase 0 — Decisions and prerequisites

- [ ] Record Kubernetes versions, source/product cluster CIDRs, cloud/network provider, ingress controller, and storage classes.
- [ ] Prove private routing and DNS from every source cluster to a temporary internal product endpoint.
- [ ] Select TLS and workload-authentication approach, tenant model, and Grafana SSO provider.
- [ ] Estimate telemetry volume and retention; select object store and queue service.
- [ ] Lock compatible chart/app versions. Confirm current Mimir ingest-storage and Tempo Kafka requirements and Loki chart migration notes.
- [ ] Decide whether Fluentd transformations or routing justify the extra source-cluster hop.

**Exit criteria:** written version matrix, route/DNS/TLS design, tenant model, storage/queue decision, and capacity estimate.

## Phase 1 — Product-cluster foundation

- [ ] Create namespaces, service accounts, RBAC, quotas, node pools, storage classes, and network policies.
- [ ] Deploy private object storage access, buckets/prefixes, credentials, encryption, and lifecycle/retention controls.
- [ ] Deploy the selected Kafka-compatible service and isolated topics if required by the chosen chart versions.
- [ ] Deploy internal ingress/gateway with TLS, authentication, authorization, tenant mapping, and rate limits.
- [ ] Add synthetic health checks from the source networks.

**Exit criteria:** no public exposure; authenticated test requests reach only approved internal endpoints; data services have persistent storage and health checks.

## Phase 2 — Core observability backends

- [ ] Install Mimir distributed and configure remote-write tenant routing and object storage.
- [ ] Install Loki and configure write/query gateway, storage, retention, and compaction.
- [ ] Install Tempo in the selected topology, configure OTLP receivers, object storage, and required queue.
- [ ] Install Grafana with OIDC/SSO, provision all three datasources, and restrict datasource administration.
- [ ] Add stack self-monitoring and independent alerts for ingress, storage, queues, PVs, and component health.

**Exit criteria:** authenticated smoke-test metrics, logs, and traces are queryable in Grafana with correct tenant and cluster attribution.

## Phase 3 — One source-cluster pilot

- [ ] Install Fluent Bit and Fluentd; verify metadata, redaction, buffering, and Loki delivery.
- [ ] Install Prometheus/Alloy; verify scrape targets, relabeling, remote-write queue, and Mimir delivery.
- [ ] Install OpenTelemetry Collectors; instrument one service, verify OTLP delivery, and correlate traces to logs/metrics.
- [ ] Test network outage, ingress failure, queue saturation, and recovery without exhausting node/PV storage.
- [ ] Create initial cluster, service, and pipeline dashboards and alerts.

**Exit criteria:** pilot data appears within agreed latency, source buffers recover after an outage, and all telemetry has a stable `cluster` identity.

## Phase 4 — Production hardening

- [ ] Load-test expected peak ingestion and query workloads; tune replicas, limits, retention, cardinality, and ingress throttles.
- [ ] Validate HA behavior, pod disruption, zone/node failures, object-store restore, and queue recovery.
- [ ] Document SLOs, retention, RPO/RTO, runbooks, access reviews, secret rotation, and upgrade/rollback process.
- [ ] Pin all charts and images; automate GitOps promotion and policy checks.

**Exit criteria:** capacity and recovery tests meet agreed SLO/RPO/RTO; alerts and ownership are documented.

## Phase 5 — Multi-cluster rollout

- [ ] Roll out one source cluster at a time using cluster-specific labels, credentials, and quotas.
- [ ] Validate tenant isolation and dashboards after each onboarding.
- [ ] Track ingestion volume, dropped data, cost, and cardinality per cluster.
- [ ] Review capacity and retention monthly; upgrade through a staging environment first.

**Exit criteria:** each cluster has an owner, healthy pipeline, isolated identity, dashboards, and tested support path.

## Delivery order

`Private network + identity → object storage / queue → ingress gateway → Mimir/Loki/Tempo → Grafana → one source pilot → resilience tests → remaining clusters`
