# Flux demo

This repository contains a small Flux GitOps setup for a local Kubernetes cluster running on k3s.

## What is included

- A simple whoami sample application.
- Monitoring stack with:
  - prometheus-community
  - Loki
  - Grafana
- Ingress for Grafana at grafana.localhost using traefik.
- Declarative secret management using Sealed Secrets.

## Repository layout

- apps/whoami: sample application manifests
- monitoring: HelmRepository, HelmRelease, and SealedSecret definitions
- ingress: ingress resource for Grafana
- kustomization.yaml: top-level Flux entrypoint

## Current status

The repo is configured to reconcile the following resources through Flux/Kustomize:

- whoami deployment and service
- monitoring stack with reduced resource requests for a smaller VM
- Grafana admin credentials stored as a SealedSecret rather than plaintext in Git


## Secret management

Grafana credentials are managed through a SealedSecret in monitoring/grafana-admin-sealed.yaml. The Sealed Secrets controller in the cluster decrypts it into a normal Kubernetes Secret at apply time.

```