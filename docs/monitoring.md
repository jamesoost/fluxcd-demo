# Monitoring

This page explains what observability coverage exists today, why the stack is designed this way, and where the blind spots are.

## Observability Goal

The goal of the observability implementation provide enough visibility to answer three core questions:

- Is the cluster healthy?
- Are platform services available?
- Are logs accessible per namespace for rapid triage?

## Implemented Stack

The monitoring namespace includes three HelmReleases defined in [infrastructure/monitoring/releases.yaml](../infrastructure/monitoring/releases.yaml):

- kube-prometheus-stack - A pre-packaged, monitoring bundle for Kubernetes. Used for visibility into cluster health.
- loki - cost-effective log aggregation system. Used for log visibility. 
- grafana - A visualization and dashboard platform. Used to correlate metrics and logs in one place and display them in the form of dashboards.

## Topology

- Prometheus is provided by kube-prometheus-stack with the included instance of Grafana disabled.
- Loki is provided by loki-stack with promtail enabled.
- Grafana runs as a standalone release (not the embedded chart variants from other stacks).

Design decision:
- Standalone Grafana was chosen to keep dashboard and datasource provisioning predictable and independent of upstream chart defaults.
- kube-prometheus-stack is used over a standalone prometheus implementation because it includes Prometheus Operator without which scraping would need to be configured manually.

## Datasource Provisioning

Grafana datasources are provisioned from:
- infrastructure/monitoring/datasources/datasources.yaml

Configured datasources:
- Prometheus: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
- Loki: http://loki.monitoring.svc.cluster.local:3100

Reference: [infrastructure/monitoring/datasources/datasources.yaml](../infrastructure/monitoring/datasources/datasources.yaml)

## Dashboard Provisioning

Dashboards are loaded from files under:
- infrastructure/monitoring/dashboards/

Kustomize generates ConfigMaps with fixed names and labels. Grafana sidecars discover and import them based on labels:
- grafana_dashboard: "1"
- grafana_datasource: "1"

Reference: [infrastructure/monitoring/kustomization.yaml](../infrastructure/monitoring/kustomization.yaml)

## Grafana Credentials

Grafana admin credentials come from secret grafana-admin, which is synced through External Secrets from the vault.

> If login fails, re-check Vault bootstrap flow in [vault.md](vault.md).

## Coverage and Gaps

Current coverage:
- Cluster and workload health.
- Centralized logs across namespaces.
- Datasource and dashboard bootstrapping through Git-managed manifests.

Current gaps:
- No long-term metrics retention backend.
- No long-term log storage backend.
- No Dagster-specific RED dashboard set yet.

## Useful Checks

```bash
flux get helmreleases -n monitoring
kubectl -n monitoring get pods
kubectl -n monitoring get configmaps
kubectl -n monitoring get externalsecret grafana-admin
kubectl -n monitoring get secret grafana-admin
```

Pass criteria:
- HelmReleases are ready.
- Grafana admin secret exists.
- Datasource and dashboard ConfigMaps exist.

Ingress endpoint:

```bash
kubectl -n monitoring get ingress
```

## Common Issues

- Datasource missing: confirm grafana-datasources ConfigMap exists and has grafana_datasource label.
- Dashboard missing: confirm dashboard ConfigMap exists and has grafana_dashboard label.
- Grafana startup issues: inspect probes and pod events.

```bash
kubectl -n monitoring describe pod -l app.kubernetes.io/name=grafana
kubectl -n monitoring logs deploy/grafana
```

## Trade-offs

- Lightweight stack is easy to run on a local node.
- Retention and high-cardinality behavior are intentionally constrained to match with the setups limitations.

## Next Reads

- System design: [architecture.md](architecture.md)
- Secret and credential flow: [vault.md](vault.md)
- Failure recovery index: [troubleshooting.md](troubleshooting.md)
- CI validation strategy: [ci.md](ci.md)
- Production scaling path: [scaling.md](scaling.md)
