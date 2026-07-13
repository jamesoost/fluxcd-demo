# Troubleshooting

Runbook format: symptom, likely cause, verify, fix, prevent.

## Runbook: Flux Reconciliation Failing

**Symptom:**
- Kustomization not ready
- Resources not updating after commit

**Likely causes:**
- Invalid rendering in one environment path.
- Source or controller reconciliation errors.

**Verify:**

```bash
flux get kustomizations -A
flux logs --level=error
kubectl get events -A --sort-by=.lastTimestamp
```

**Fix:**
- Correct the invalid manifest or patch and re-apply through Git.
- Force reconcile after correction.

```bash
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
```

**Prevent:**
- Run local render checks before opening a PR.
- Keep overlays focused and minimal.

## Runbook: HelmRelease Not Ready

**Symptom:**
- HelmRelease stays non-ready
- Pods crashloop or remain pending

**Likely causes:**
- Invalid Helm values.
- Missing dependent secret.
- Resource pressure in the cluster.

**Verify:**

```bash
flux get helmreleases -A
kubectl -n dagster describe helmrelease dagster
kubectl -n dagster describe helmrelease dagster-postgresql
kubectl -n monitoring describe helmrelease grafana
kubectl -n monitoring describe helmrelease kube-prometheus-stack
kubectl -n monitoring describe helmrelease loki
kubectl top nodes
kubectl get events -A --field-selector reason=FailedScheduling
```

**Fix:**
- Correct values in the related HelmRelease manifest.
- Ensure dependency secrets exist.
- If pods are unschedulable due to resource pressure, reduce requested resources in the overlay patch for this environment, or free up cluster capacity.
- Reconcile and watch events.

**Prevent:**
- Keep changes small and scoped.
- Validate rendered manifests in CI and locally.
- Keep resource requests realistic for the local single-node baseline (see [Prerequisites](getting-started.md#prerequisites)).

## Runbook: ExternalSecret Not Syncing

**Symptom:**
- Target secret does not appear
- ExternalSecret shows provider/auth errors

**Likely causes:**
- Missing or expired vault-token secret.
- Vault policy path mismatch.
- Incorrect remoteRef key or property mapping.

**Verify:**

```bash
kubectl -n dagster get externalsecret dagster-postgresql
kubectl -n monitoring get externalsecret grafana-admin
kubectl -n dagster describe externalsecret dagster-postgresql
kubectl -n monitoring describe externalsecret grafana-admin
kubectl -n dagster get secret vault-token
kubectl -n monitoring get secret vault-token
```

Verify Vault paths and properties match expected keys:
- secret/dagster/postgres -> password, admin-password
- secret/grafana -> admin-user, admin-password

**Fix:**
- Recreate vault-token with a fresh app-scoped token.
- Correct Vault policy path and ExternalSecret mapping.
- Re-run sync checks.

**Prevent:**
- Keep one policy and token per app namespace.
- Document secret mapping changes in PR descriptions.

## Runbook: Vault Is Sealed

**Symptom:**
- ExternalSecret reads fail
- Vault health indicates sealed state

**Likely causes:**
- Pod restart or manual reseal event.

**Verify:**

```bash
kubectl -n secrets-infrastructure get pods
kubectl -n secrets-infrastructure exec -it vault-0 -- vault status
```

If sealed, unseal with threshold keys:

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- vault operator unseal '<UNSEAL_KEY>'
```

Repeat until unsealed.

**Fix:**
- Re-run app sync checks after unseal.

**Prevent:**
- For production, move to auto-unseal design described in [scaling.md](scaling.md).

## Runbook: Grafana Datasources or Dashboards Missing

**Symptom:**
- No Prometheus/Loki datasource in UI
- Dashboard not auto-imported

**Likely causes:**
- Missing labels on generated ConfigMaps.
- Sidecar import issues or pod restart drift.

**Verify:**

```bash
kubectl -n monitoring get configmaps
kubectl -n monitoring get configmap grafana-datasources -o yaml
kubectl -n monitoring get pods
kubectl -n monitoring logs deploy/grafana
```

Confirm labels exist on generated ConfigMaps:
- grafana_datasource: "1"
- grafana_dashboard: "1"

**Fix:**
- Correct label configuration and regenerate manifests.
- Restart Grafana deployment if sidecar import is stuck.

```bash
kubectl -n monitoring rollout restart deploy/grafana
```

**Prevent:**
- Keep datasource and dashboard files managed in Git and reviewed with each update.

## Runbook: Pods Unhealthy After Fresh Bootstrap

**Symptom:**
- Dagster, PostgreSQL, or Grafana stay unhealthy after bootstrap.

**Likely cause:**
- Secrets were not bootstrapped yet, so dependent pods cannot start.
- Incorrect application configuration (e.g. wrong image tag, bad Helm values) causing the pod to fail independent of secrets.

**Verify:**

```bash
kubectl -n dagster get pods
kubectl -n monitoring get pods
```

If a pod is `Pending` or stuck waiting on a volume/secret mount, it's likely the missing-secret case. If a pod is `CrashLoopBackOff` or `Error`, check its logs and events to distinguish a config problem from a secret problem:

```bash
kubectl -n <namespace> describe pod <pod-name>
kubectl -n <namespace> logs <pod-name>
```

**Fix:**
- If the pod is waiting on a secret: complete the secret bootstrap flow in [vault.md](vault.md).
- If the pod is crash-looping with an application-level error in its logs: correct the relevant HelmRelease values and let Flux reconcile.

**Prevent:**
- Treat secret bootstrap as a required phase in onboarding, not an optional step.
- Validate rendered Helm values in CI before merge (see [ci.md](ci.md)).

## Next Reads

- Setup flow: [getting-started.md](getting-started.md)
- Secret lifecycle and recovery: [vault.md](vault.md)
- Monitoring internals: [monitoring.md](monitoring.md)
- CI validation strategy: [ci.md](ci.md)
- Production hardening: [scaling.md](scaling.md)