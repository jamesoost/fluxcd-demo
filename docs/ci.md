# CI Validation

This page documents the quality gate strategy for this project and why each check exists.
The core concept behind the current CI implementation is to automatically validate Kubernetes and GitOps manifests whenever code is pushed or a pull request is opened. It runs a security and best-practice linting check on the output, reporting any configuration errors or vulnerabilities directly back to the pull request summary.

Workflow source: [.github/workflows/gitops-pr-validation.yml](../.github/workflows/gitops-pr-validation.yml)

## Validation Goal

Catch structural manifest errors before merge while balancing strictness and contributor velocity.

The CI pipeline does this by:

- Prevent Deployment Failures: It guarantees that manifests are syntactically valid YAML and match official Kubernetes and Custom Resource (CRD) schemas, preventing the GitOps controller (like Flux) from getting stuck on invalid code.

- Enforce Security and Best Practices: It automatically flags risky configurations, like running containers as root, missing resource limits, or omitting health checks before they can be merged.

- Ensure Drift-Free Manifest Generation: It proves that Kustomize overlays and Helm releases can actually be rendered and compiled successfully without errors.

## Pipeline Stages

1. Build manifests for clusters/dev and clusters/prod via kustomize.
2. Validate rendered YAML with kubeconform, including Flux CRDs.
3. Lint rendered app manifests with kube-linter.
4. Render monitoring Helm charts with helm template.
5. Lint rendered monitoring charts with kube-linter (advisory).
6. Publish lint summary in GitHub step summary.

## Blocking vs Advisory

The CI pipeline splits the app linting and monitoring linting into two seperate steps.

- App kube-linter - Checks the application configuration files. This stage is enforced failing the workflow and blocking merges unless issues are remediated.
- Monitoring kube-linter- Checks the monitoring configuration files. This stage is advisory and summarised, but does not block merge by itself.

Decision rationale:
- App manifests represent the core deployment path and are treated as merge-critical.
- Monitoring lint still surfaces risks, but is non-blocking to avoid reducing delivery speed for non-core chart findings.

## Trust Boundaries

- CI proves manifests render and validate syntactically.
- CI does not prove runtime correctness in your cluster.
- Cluster-specific behavior still requires reconciliation and runtime checks.
- CI does not verify that documentation, such as the App Mappings table in [vault.md](vault.md#app-mappings-in-this-repo), stays aligned with the actual ExternalSecret manifests. This mapping is maintained by hand and can drift from the manifests if one is updated without the other.

## Why This Matters

The split between blocking and advisory checks preserves merge protection for core app manifests while still surfacing monitoring chart quality issues for follow-up.

Example regression classes this pipeline can catch:
- Invalid Kubernetes schema fields after chart or API changes.
- Missing required fields in rendered resources.
- Lint-identified risky workload patterns in app manifests.

## Reproduce Locally

Render both environments:

```bash
kustomize build clusters/dev > /tmp/rendered-dev.yaml
kustomize build clusters/prod > /tmp/rendered-prod.yaml
cat /tmp/rendered-dev.yaml /tmp/rendered-prod.yaml > /tmp/rendered.yaml
```

Schema validation example:

```bash
kubeconform \
  -strict \
  -summary \
  -ignore-missing-schemas \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  /tmp/rendered-dev.yaml \
  /tmp/rendered-prod.yaml
```

Lint apps:

```bash
kube-linter lint /tmp/rendered.yaml
```

## Trade-offs

- Strong static checks reduce avoidable deployment failures.
- Advisory monitoring lint can let non-critical issues through and depends on team discipline for follow-up.

## Next Reads

- First deployment workflow: [getting-started.md](getting-started.md)
- Architecture and layout context: [architecture.md](architecture.md)
- Troubleshooting failing reconciliations: [troubleshooting.md](troubleshooting.md)
