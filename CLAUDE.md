# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Flux GitOps repo for a single-node Talos Linux Kubernetes homelab. Flux watches this repo and reconciles all resources automatically. There is no build step, no CI/CD, no tests — changes are applied by pushing to `main`.

**Private companion repo** (`corruptmane/homelab-private`) holds OpenTofu configs, Talos machine patches, and the operational runbook.

## Workflow

```bash
# Validate YAML before pushing
kubectl kustomize infrastructure/controllers --enable-helm  # dry-run controllers
kubectl kustomize infrastructure/configs                     # dry-run configs

# Force Flux to pick up changes immediately (otherwise 1h poll interval)
flux reconcile kustomization flux-system --with-source

# Watch reconciliation
flux get helmreleases -A --watch
flux events --for HelmRelease/<name> -n <ns> --watch

# Debug a stuck HelmRelease
flux logs --follow --level=info
helm get values <release> -n <ns> --all
kubectl events -n <ns> --types=Warning
```

## Repo Structure & Dependency Model

```
clusters/homelab/
  flux-system/                        # Flux bootstrap — DO NOT EDIT
  infrastructure-controllers.yaml     # Kustomization CR → infrastructure/controllers (wait: true)
  infrastructure-configs.yaml         # Kustomization CR → infrastructure/configs (dependsOn: controllers)

infrastructure/
  controllers/                        # HelmRepos + Namespaces + HelmReleases (operators/agents)
    kustomization.yaml                # Lists all controller subdirectories
    cilium/ cert-manager/ external-secrets/ external-dns/
    kyverno/ longhorn/ cnpg/ monitoring/
  configs/                            # Post-deploy resources (ClusterIssuers, ExternalSecrets, PVs)
    kustomization.yaml                # Lists all config subdirectories
    cluster-secret-store.yaml
    cert-manager/ cnpg/ nfs/
```

**Dependency chain:** Flux deploys `infrastructure-controllers` first (with `wait: true`), then `infrastructure-configs` only after all controllers are Ready. Adding a new component means adding its directory AND listing it in the parent `kustomization.yaml`.

## Conventions

### HelmRelease Pattern
Every HelmRelease follows this structure — maintain consistency:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <app>
  namespace: <target-namespace>   # NOT flux-system — must match where pods run
spec:
  interval: 1h
  chart:
    spec:
      chart: <chart>
      version: "<pinned-semver>"
      sourceRef:
        kind: HelmRepository
        name: <repo>
        namespace: flux-system    # All HelmRepos live here
      interval: 24h
  install:
    remediation:
      retries: 3
  upgrade:
    cleanupOnFail: true
    remediation:
      retries: 3
  values: { ... }
```

### HelmRepository Pattern
All HelmRepositories use `namespace: flux-system` with `interval: 24h`.

### Namespace Pattern
Namespaces that run privileged workloads (Longhorn, monitoring) need PSA labels:
```yaml
labels:
  pod-security.kubernetes.io/enforce: privileged
  pod-security.kubernetes.io/audit: privileged
  pod-security.kubernetes.io/warn: privileged
```

### Secrets
Never commit secrets. All secrets flow through: **AWS SSM → ExternalSecret → Kubernetes Secret**.
- ClusterSecretStore `aws-ssm` is the single entry point (region: `eu-central-1`)
- SSM parameter paths always start with `/homelab/` (leading slash required)
- ExternalSecrets use API version `external-secrets.io/v1` (not v1beta1)

## Cluster-Specific Context

- **Single physical host** — Longhorn runs 1 replica (replication is pointless), CSI sidecar replicas capped at 2
- **Talos Linux** — no systemd, no SSH; kube-scheduler/controller-manager/etcd/kube-proxy metrics endpoints are not exposed; Cilium replaces kube-proxy
- **Gateway API** — used instead of Ingress everywhere; external-dns sources are `gateway-httproute`, `gateway-grpcroute`, `gateway-tlsroute`
- **Cilium must set `cluster.name: homelab`** — omitting this breaks Hubble relay TLS (certs encode cluster name)
- **VM Operator webhooks must stay disabled** — `admissionWebhooks.enabled: false` to avoid race condition where webhooks block CR creation during install

## Monitoring Architecture

Dual-stack with fan-out collectors:
```
VMAgent ──remote-write──→ VMSingle (10Gi) + Prometheus (5Gi)
Vector DaemonSet ────────→ VictoriaLogs (5Gi) + Loki (3Gi)
OTel Collector ──────────→ VictoriaTraces (5Gi) + Tempo (3Gi)
```
Victoria stack is primary; Grafana-ecosystem backends are for comparison/learning. Grafana UI is not deployed yet (Layer 4).
