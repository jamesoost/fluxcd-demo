# Vault and Secrets

This page covers how application credentials are managed in this project, the reasoning behind that design, and the exact runbook used to bootstrap, recover, and rotate secrets.

## Security Goal

Keep app credentials out of Git entirely and deliver them to the namespaces that need them in a controlled, auditable way, with no credential accessible outside the app it was issued for.

## Why This Design

- Decision: use Vault as the single source of truth for secret values, with the External Secrets controller responsible for syncing them into Kubernetes as native Secrets.
- Why: this keeps secrets runtime managed rather than committed to Git, scopes read access to one Vault policy per app, and limits each synced secret to the namespace that app runs in, so a compromise in one namespace does not expose credentials belonging to another.
- Trade-off: this adds real operational overhead compared to static Kubernetes Secrets checked into Git, since each app now needs its own Vault path, policy, and token issued and maintained by hand in this setup, rather than a secret simply existing wherever it is referenced. See [scaling.md](scaling.md) for how this setup could evolve toward less manual overhead, such as automated secret provisioning or a managed cloud secrets platform.

## What Runs in This Repo

- Vault is deployed in namespace secrets-infrastructure.
- External Secrets controller is deployed in namespace secrets-infrastructure and depends on Vault.
- App namespaces contain SecretStore and ExternalSecret resources that read from Vault.

References:

- Vault release: [infrastructure/secrets-infrastructure/vault-release.yaml](../infrastructure/secrets-infrastructure/vault-release.yaml)
- External Secrets controller: [infrastructure/secrets-infrastructure/external-secrets-release.yaml](../infrastructure/secrets-infrastructure/external-secrets-release.yaml)
- Dagster ExternalSecret: [apps/dagster/external-secrets-dagster.yaml](../apps/dagster/external-secrets-dagster.yaml)
- Grafana ExternalSecret: [infrastructure/monitoring/external-secrets-grafana.yaml](../infrastructure/monitoring/external-secrets-grafana.yaml)

**Current Vault profile in this demo:**
- Single replica, HA disabled.
- Manual unseal required after restart/reseal.
- Data stored in a single PVC.

## Threat Model Assumptions

- **Git repository is not a secret store.** No credential, token, Git history is effectively permanent. This is why every secret in this repo is written directly to Vault via `kubectl exec`, never through a manifest.
- **Cluster namespaces are semi-trusted boundaries.** Namespace isolation limits which workloads can reach which Kubernetes Secrets by default, but it is not a hard security boundary. Anyone with `kubectl exec` or `get secret` access to a namespace can read what is stored there, so namespace separation contains blast radius, it does not prevent access by an operator who already has cluster credentials.
- **Vault token leakage should be contained by app-scoped policy design.** Each app receives its own Vault token, scoped to a policy that can only read that app's own secret path. If one app's token is compromised, the attacker can read that app's secret and nothing else in Vault, rather than every secret in the system.

These assumptions describe what this design protects against (accidental Git exposure, cross-app secret access from a single leaked token) and what it does not (a compromised cluster operator, or a compromised pod within the same namespace as its own secret).

## First-Time Initialization

Initialize once:

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- vault operator init
```

Store init output securely. It contains:
- Unseal keys
- Initial root token

Unseal using any 3 keys:

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- vault operator unseal '<UNSEAL_KEY>'
```

Repeat until Vault reports unsealed.
> Notes:
> - An admin token should be generated and the root token revoked as the root token gives unrestricted access to the whole vault.
> - Unsealing can also be done on the vaults UI which you can get the endpoint for by running `kubectl get endpoints -n secrets-infrastructure `

## Recovery Scenarios and Risk

- If the vault seals and you have at least 3 unseal keys, you can run unseal and continue.
- Lost root token but have unseal keys: use `vault operator generate-root`. This uses a quorum of unseal keys plus an OTP or PGP key and does not require an existing Vault token.
- Lost unseal keys, but Vault is still unsealed and running: it will keep functioning normally until the next reseal or restart. Read out and back up anything you need from it now, since you will need to reconstruct Vault from scratch once it stops running.
- Lost unseal keys, and Vault is sealed or about to restart: there is no recovery path. Proceed to the last-resort reset below.

Last-resort reset (data loss):

```bash
kubectl -n secrets-infrastructure delete pod vault-0
kubectl -n secrets-infrastructure delete pvc data-vault-0
```

Then initialize again and re-bootstrap app secrets.

> Manual unseal is acceptable for this local demo but is a known production gap. See [scaling.md](scaling.md).

## Secret Sync Pattern

Each app follows the same model:
1. Write secret data in Vault.
2. Create scoped read policy.
3. Issue token for that policy.
4. Place token in app namespace as secret named vault-token.
5. ExternalSecret syncs target Kubernetes secret.

Before starting, confirm the namespace, SecretStore, and ExternalSecret already exist.

## Generic Bootstrap Steps

1. Login:

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- vault login <VAULT_ADMIN_TOKEN>
```

2. Enable KV v2 (one-time per cluster):

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- vault secrets enable -path=secret kv-v2
```

3. Write app secret:

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- \
  vault kv put secret/<APP_PATH> <KEY>=<VALUE> <KEY2>=<VALUE2>
```

4. Create scoped read policy:

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- sh -lc "cat <<'EOF' > /tmp/eso-<APP_NAME>.hcl
path \"secret/data/<APP_PATH>\" {
  capabilities = [\"read\"]
}
EOF
vault policy write eso-<APP_NAME> /tmp/eso-<APP_NAME>.hcl"
```

5. Create policy token and copy auth.client_token:

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- \
  vault token create -policy=eso-<APP_NAME> -format=json
```

6. Create vault-token secret in app namespace:

```bash
kubectl -n <APP_NAMESPACE> create secret generic vault-token \
  --from-literal=token='<CLIENT_TOKEN>'
```

7. Verify sync:

```bash
kubectl -n <APP_NAMESPACE> get externalsecret
kubectl -n <APP_NAMESPACE> describe externalsecret <EXTERNALSECRET_NAME>
kubectl -n <APP_NAMESPACE> get secret <TARGET_SECRET_NAME>
```

`<EXTERNALSECRET_NAME>` and `<TARGET_SECRET_NAME>` come from the app's own ExternalSecret manifest (see the [App Mappings table](#app-mappings-in-this-repo) below for the two apps already in this repo, or the app's `external-secrets-*.yaml` file for a new one).

Verification target: the generated secret name must match what the consuming HelmRelease expects.

## App Mappings In This Repo

| App | Vault path | Properties | Policy | Namespace | ExternalSecret | Target Secret |
| --- | --- | --- | --- | --- | --- | --- |
| Grafana | secret/grafana | admin-user, admin-password | eso-grafana | monitoring | grafana-admin | grafana-admin |
| Dagster PostgreSQL | secret/dagster/postgres | password, admin-password | eso-dagster | dagster | dagster-postgresql | dagster-postgresql |

Ensure these mappings stay aligned with the ExternalSecret manifests.

## Grafana Example

```bash
kubectl -n secrets-infrastructure exec -it vault-0 -- vault login <VAULT_ADMIN_TOKEN>

kubectl -n secrets-infrastructure exec -it vault-0 -- \
  vault kv put secret/grafana admin-user=admin admin-password='<STRONG_PASSWORD>'

kubectl -n secrets-infrastructure exec -it vault-0 -- sh -lc "cat <<'EOF' > /tmp/eso-grafana.hcl
path \"secret/data/grafana\" {
  capabilities = [\"read\"]
}
EOF
vault policy write eso-grafana /tmp/eso-grafana.hcl"

kubectl -n secrets-infrastructure exec -it vault-0 -- \
  vault token create -policy=eso-grafana -format=json

kubectl -n monitoring create secret generic vault-token \
  --from-literal=token='<CLIENT_TOKEN>'

kubectl -n monitoring get externalsecret grafana-admin
kubectl -n monitoring get secret grafana-admin
```

## Trade-offs

- App-scoped policies reduce blast radius. If one app's token leaks, the attacker can only read that app's own secret path, not any other app's credentials or Vault's admin functions.
- Per-app tokens add setup overhead. Onboarding a new app means manually writing its secret, creating a scoped policy, and issuing a token before its pods can start, rather than a secret being available immediately as part of the app's own manifests. See [ci.md](ci.md) for the related gap: this manual bootstrap isn't checked by CI, so it can drift from the ExternalSecret manifests if one is updated without the other.
- Manual key custody is riskier since there is no rotation policy or audit trail for who holds or uses the unseal keys, but it keeps the workflow explicit for learning. In a real production setup this would be replaced by auto-unseal and centrally managed key custody, see [scaling.md](scaling.md).

## Security Notes

- Never commit Vault tokens or generated passwords to Git, even temporarily or in a commit that gets reverted later, since Git history is effectively permanent.
- Keep policies and tokens app-scoped. A single shared admin token used across every app would mean one leak exposes every secret in the system, rather than just one app's.
- Rotate app tokens on any suspected leak. Revoking a compromised token and issuing a new one via the [Generic Bootstrap Steps](#generic-bootstrap-steps) is sufficient, there is no need to rotate the underlying Vault unseal keys or root token unless those themselves were exposed.

## Next Reads


- Platform setup flow: [getting-started.md](getting-started.md)
- Monitoring credentials usage: [monitoring.md](monitoring.md)
- Common failures and recovery: [troubleshooting.md](troubleshooting.md)
- CI validation strategy: [ci.md](ci.md)
- Production scaling path: [scaling.md](scaling.md)
