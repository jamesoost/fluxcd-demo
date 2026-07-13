# Scaling

This project is currently scoped as a local demo. To move this implementation to production, several scaling and reliability limitations would need to be address .

## Current State
 
 The current state of this deployment have the following major limitations:

- A Single-node cluster - No high availability or node-level failover.
- A Single-replica Vault - Manual unseal required, creating a single point of failure.
- A Local small retention model for metrics and logs - Minimal retention for metrics and logs, unsuitable for long-term auditing.

## Evolution Framework

When scaling this project use of a staged/Three-Horizon Model to reduce migration risk would be recommended.

- Stage 1(Now): Maintain local reproducibility and strict GitOps workflows.
- Stage 2(Next): Harden platform security, automate Vault unsealing, and establish high availability.
- Stage 3(Later): Scale throughput, implement multi-cluster operations, and enforce global governance policies.

## Compute and Cluster Topology

A robust and reliable platform requires upgrading the underlying compute and cluster architecture to ensure high availability and proper environment isolation.

- Migrate to multi-node Kubernetes: Move workloads from a single-node VM to a multi-node cluster.
- Deploy a high-availability control plane: Use an odd number of control-plane nodes to maintain quorum, or transition to a managed Kubernetes service (e.g., EKS, GKE, AKS).
- Establish staging environments: Create a staging and testing environment that mirrors production scaling ratios to accurately test performance under load.
- Isolate environments: Use physically separate clusters (or strongly isolated node pools with taints and tolerations) to divide development, staging, and production workloads.

*Why*: Stable cluster primitives and clear failure-domain separation are prerequisites for all other scaling and security improvements.

## Flux Topology

To scale our GitOps delivery model, structure the repository layout and deployment pipelines to support multi-environment and multi-cluster rollouts.

- Maintain one reconciliation path per environment: Keep environment configurations isolated within the Git repository to prevent accidental cross-environment updates.
- Isolate multi-cluster bootstraps: For multi-cluster expansion, map each new cluster to its own Git path and dedicated promotion strategy to avoid configuration drift.
-  Implement strict environment gates: Add automated checks, manual approvals, or pull-request gates before syncing changes to production.

Reference entrypoints:
- [clusters/dev/kustomization.yaml](../clusters/dev/kustomization.yaml)
- [clusters/prod/kustomization.yaml](../clusters/prod/kustomization.yaml)

## Vault Hardening

Current demo state:
- Single Vault pod
- High Availability (HA) disabled
- Manual unseal required

Production direction:
- Deploy Vault in HA mode: Run Vault in High Availability using integrated Raft storage or a supported external backend.
- Enable auto-unseal: Configure auto-unseal using a cloud Key Management Service (KMS) like AWS KMS, Azure Key Vault, or GCP KMS.
- Formalize secrets governance: Implement strict key custody, automate token rotation, and document clear break-glass procedures for emergency access.
- Establish backup routines: Implement and regularly test automated backup and restore procedures.

## Storage

To prevent data loss and support stateful workloads, the storage layer must transition from ephemeral local disks to resilient, enterprise-grade storage solutions.

- Implement durable CSI drivers: Replace local-path storage provisioners with durable, cloud-native Container Storage Interface (CSI) drivers (such as AWS EBS or Azure Disk) to decouple storage from node lifecycles.
- Scale database storage: Right-size PostgreSQL persistence, IOPS, and storage capacity based on your production workload profiles.
- Automate snapshots and recovery: Set up automated volume snapshots and regularly run disaster recovery drills for all stateful components.

Reference for current persistence baseline:
- [apps/dagster/postgres-release.yaml](../apps/dagster/postgres-release.yaml)
- [overlays/prod/patches/postgres-release.yaml](../overlays/prod/patches/postgres-release.yaml)

## Monitoring at Scale

To handle production-level telemetry without degrading system performance or incurring excessive cloud costs, optimise the observability stack for high throughput and long-term storage.

- Control Prometheus cardinality and retention: Implement strict metric cardinality controls and tune data retention windows to prevent Prometheus from running out of memory.
- Configure Prometheus remote-write: Offload metrics to a dedicated long-term storage solution (such as Thanos, Cortex, or Grafana Mimir) using Prometheus remote-write.
- Transition Loki to object storage: Move Loki’s storage backend from expensive block storage to cost-effective, scalable cloud object storage (like AWS S3, Azure Blob, or Google Cloud Storage) as log volume increases.

> set retention periods on all of the long term storage solutions to manage costs while still meeting practical, audit and compliance requirements 

Reference baseline: [infrastructure/monitoring/releases.yaml](../infrastructure/monitoring/releases.yaml)

## Networking and Ingress

To ensure reliable external access, high availability, and secure transit, upgrade the ingress path from local exposure to a production-grade routing architecture.

- Deploy cloud load balancers: Replace single-node port forwarding or node-port exposure with a highly available, cloud-managed LoadBalancer service to distribute incoming traffic across healthy nodes.
- Automate DNS and TLS lifecycles: Integrate automated DNS provisioning (such as external-dns) and automated TLS certificate management (such as cert-manager with Let's Encrypt or your enterprise CA) to secure traffic without manual intervention.

## CI and Delivery

To safely scale deployment velocity, transition the delivery pipeline from simple syntax validation to automated compliance and post-deployment validation.

- Preserve existing validation rules: Retain the current code schema validation and linting checks as the first line of defense in the pull request pipeline.
- Enforce environment-specific policies: Run automated policy-as-code checks before promoting configurations to production to ensure compliance with security and operational standards.
- Implement drift detection and SLOs: Deploy automated drift-detection dashboards to catch manual cluster changes, and establish Service Level Objectives (SLOs) to measure release health and rollback stability.

## Recommended Upgrade Sequence

To minimize risk and ensure a stable transition, execute the migration in the following order:

1. Establish the Multi-Node Foundation: Build and configure the multi-node cluster topology first. You must have a highly available compute layer before deploying resilient services.
2. Harden Secrets and Identity: Migrate Vault to a High Availability (HA) configuration with auto-unseal. Hardening the identity provider early ensures all subsequent platform components can securely fetch credentials.
3. Scale Telemetry and Storage: Implement long-term storage and remote-write backends for metrics and logs. This ensures you have deep operational visibility before moving complex workloads.
4. Transition to Multi-Cluster GitOps: Roll out the multi-cluster reconciliation paths and automated promotion gates once the platform's core infrastructure, security, and observability layers are stable.

## Trade-offs

- Incremental Hardening - Implementing changes in phases reduces overall delivery risk but requires maintaining hybrid states.
  - Pros: Lower migration risk, easier rollback paths, and faster feedback loops. If an issue occurs during a phase, you only have to troubleshoot one subsystem rather than a completely overhauled stack.
  - Cons: You must temporarily support a hybrid architecture where some components are production-grade while others still rely on demo-level configurations.

- Production Security, Reliability and Operational Overhead - Adding high availability, multi-cluster GitOps, and auto-unsealing Vault instances eliminates single points of failure but introduces ongoing platform maintenance costs.
    - Pros: Near-zero downtime, automated disaster recovery, robust security compliance, and a resilient platform that can scale to meet enterprise traffic.
    - Cons: Increased infrastructure costs. Additionally as the platform grows futher improvements and automations will need to be implemented to manage the operators cognative load.

## Next Reads

- Current architecture: [architecture.md](architecture.md)
- Current vault model: [vault.md](vault.md)
- Validation workflow: [ci.md](ci.md)
