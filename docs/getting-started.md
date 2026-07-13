# Getting Started

This guide is an outcome-based walkthrough for bringing the platform from empty cluster to healthy reconciliation. By the end, Flux will be managing this repository's state, and Dagster, monitoring, and secrets infrastructure will be visible and reconciling, though pods will not yet show a fully healthy state, since app pods depend on a manual secrets bootstrap covered separately in [vault.md](vault.md).

## Context

This platform runs on a single local VM rather than a multi-node cluster, so the design consistently favors clarity and repeatability over high availability, see [Prerequisites](#prerequisites) for the tested hardware baseline.

## Success Criteria

By the end of this guide you should have:

- Flux controllers ready.
- Environment path reconciled.
- HelmRelease resources visible for app, monitoring, and secrets infrastructure.
- Grafana and Dagster reachable via ingress, once secrets are bootstrapped.
- A clear next step to complete secrets bootstrap in [vault.md](vault.md).

## Prerequisites

- Kubernetes cluster access (k3s used for this demo).
- kubectl configured for the target cluster.
- Flux CLI installed.
- Kustomize installed.
- Helm installed (optional for local chart debugging).
- GitHub personal access token with repository permissions.

> Tested on a single local VM (Debian) with 4 vCPUs / 16GB RAM / 50GB disk.

## Step 1: Bootstrap Reconciliation

**Problem:** The cluster has no configured state source.

**Decision:** Bootstrap Flux directly to the repository environment path.

**Expected outcome:** Flux installs controllers and links the cluster to Git.

```bash
export GITHUB_TOKEN=<YOUR_GITHUB_PAT>

flux bootstrap github \
  --owner=<GITHUB_OWNER> \
  --repository=<REPO_NAME> \
  --branch=main \
  --path=clusters/dev \
  --personal
```

Use `clusters/prod` when you intentionally want production overlay behavior.

**Notes:**
- The token is needed at bootstrap time only.
- Flux automatically stores an SSH deploy key in `flux-system` for ongoing Git access.

## Step 2: Verify Core Health

**Problem:** A successful bootstrap command does not guarantee all controllers and resources are healthy.

**Verification commands:**

```bash
flux check
flux get kustomizations
flux get helmreleases -A
```

**Pass criteria:**
- flux-system
- cluster path kustomization
- Dagster and PostgreSQL HelmReleases
- Monitoring HelmReleases
- Vault and External Secrets infrastructure

## Step 3: Validate Rendered State Locally

**Decision:** Render both environments before commit to catch structural issues early.

```bash
kustomize build clusters/dev
kustomize build clusters/prod
```

## Step 4: Bootstrap Secrets

**Problem:** Dagster, PostgreSQL, and Grafana depend on ExternalSecret-synced credentials. If Vault has not been initialized and app secrets have not been bootstrapped yet, their pods will remain unhealthy.

**Action:** Complete the steps outlined in [vault.md](vault.md) now, then return here.

## Step 5: Confirm Service Exposure

**Problem:** Reconciliation succeeding doesn't mean services are reachable yet.

**Verification commands:**

```bash
kubectl -n monitoring get ingress
kubectl -n dagster get ingress
```

If Grafana prompts for credentials, use values written during the Vault bootstrap flow in [vault.md](vault.md).

## Trade-offs

This guide favors a simple, reproducible bootstrap over full automation: one command gets Flux reconciling, and secrets are bootstrapped manually and separately rather than baked into the initial sync. See [architecture.md](architecture.md) for the reasoning behind these choices.

## Teardown

**`flux uninstall`** removes the Flux controllers and their CRDs from the cluster. It does **not** delete anything Flux deployed. Dagster, PostgreSQL, the monitoring stack, and Vault keep running, just without anything reconciling their state going forward.

```bash
flux uninstall --namespace=flux-system
```

**Optional full cleanup** removes the deployed workloads themselves by deleting their namespaces:

```bash
kubectl delete namespace dagster monitoring secrets-infrastructure
```

This deletes the namespaces and everything in them, including PVCs. However, depending on the storage class's reclaim policy, the underlying PersistentVolumes (Vault's data, PostgreSQL's data) may not be deleted automatically. Check `kubectl get pv` afterward if you want a fully clean slate, and delete any leftover volumes manually.

If you're discarding the whole VM rather than reusing it, you can skip both steps above and just destroy the VM directly.

## Next Reads

- Secrets bootstrap and runbook: [vault.md](vault.md)
- Design rationale: [architecture.md](architecture.md)
- Monitoring stack details: [monitoring.md](monitoring.md)
- If something fails: [troubleshooting.md](troubleshooting.md)

See [architecture.md](architecture.md#verification-anchors) for file-level references verifying this guide's steps.
