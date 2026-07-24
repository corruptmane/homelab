# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Flux GitOps repo for a two-node Talos Linux Kubernetes homelab (1 control-plane + 1 worker VM on a single Proxmox host). Flux watches this repo and reconciles all resources automatically. There is no build step, no CI/CD, no tests — changes are applied by pushing to `main`.

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

# Force ESO to re-sync a secret (default refreshInterval is 1h)
kubectl annotate externalsecret <name> -n <ns> force-sync=$(date +%s) --overwrite

# Unstick a failed HelmRelease (Flux won't retry on its own)
flux suspend helmrelease <name> -n <ns> && flux resume helmrelease <name> -n <ns>
```

## Repo Structure & Dependency Model

```
clusters/homelab/
  flux-system/                        # Flux bootstrap — DO NOT EDIT
  infrastructure-controllers.yaml     # Kustomization CR → infrastructure/controllers (wait: true)
  infrastructure-configs.yaml         # Kustomization CR → infrastructure/configs (dependsOn: controllers)
  infrastructure-apps.yaml            # Kustomization CR → infrastructure/apps (dependsOn: configs)

infrastructure/
  controllers/                        # HelmRepos + Namespaces + HelmReleases (operators/platform)
    kustomization.yaml
    cilium/ cert-manager/ external-secrets/ external-dns/
    longhorn/ cnpg/ monitoring/ zitadel/
  configs/                            # Post-deploy resources (ClusterIssuers, ExternalSecrets, CNPG Clusters)
    kustomization.yaml
    cluster-secret-store.yaml
    cert-manager/ cnpg/ nfs/ cilium/ zitadel/ grafana/
  apps/                               # Infrastructure apps (depend on configs-layer resources)
    kustomization.yaml
    zitadel/ grafana/
```

**Three-layer dependency chain:** Flux deploys `infrastructure-controllers` first (with `wait: true`), then `infrastructure-configs`, then `infrastructure-apps`. This avoids deadlocks where a HelmRelease in controllers would block configs from deploying resources it depends on (e.g., CNPG Cluster CRs).

A future `apps/` top-level directory (Layer 5) will hold end-user applications with `dependsOn: infrastructure-apps`.

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
- ESO refreshInterval is 1h — force sync when needed (see Workflow section)

### Gateway API Pattern
Each externally-exposed service gets: Certificate (cert-manager) + Gateway (Cilium) + HTTPRoute. external-dns auto-creates Pi-hole DNS records from HTTPRoute hostnames.

### CNPG Pattern
Each app needing PostgreSQL gets its own Cluster CR in `infrastructure/configs/<app>/`. Pin `imageName` to PG 17 (Zitadel doesn't support PG 18). Set `enableSuperuserAccess: true` if the app's init job needs superuser. CNPG auto-generates `<cluster>-app` and `<cluster>-superuser` secrets.

## SSM Parameters Required Before Deploy

These must exist in AWS SSM before the corresponding resources are deployed:

| Parameter | Required Before | How to Create |
|-----------|----------------|---------------|
| `/homelab/cloudflare-api-token` | cert-manager ClusterIssuer | Manual |
| `/homelab/pihole-password` | external-dns | Manual |
| `/homelab/cnpg-backup-access-key-id` | CNPG backup | OpenTofu |
| `/homelab/cnpg-backup-secret-access-key` | CNPG backup | OpenTofu |
| `/homelab/zitadel-master-key` | Zitadel deploy | `LC_ALL=C tr -dc A-Za-z0-9 </dev/urandom \| head -c 32` (must be exactly 32 bytes) |
| `/homelab/zitadel-admin-password` | Zitadel deploy | Must meet complexity: uppercase + lowercase + number + symbol |
| `/homelab/grafana-oidc-client-id` | Grafana deploy | From Zitadel OIDC application |
| `/homelab/grafana-oidc-client-secret` | Grafana deploy | From Zitadel OIDC application |

## Cluster-Specific Context

- **Single physical host (Beelink EQR6, 32GB RAM) running Proxmox** with two Talos VMs: control-plane `talos-cp-0` (192.168.1.30, 5GB) and worker `talos-worker-0` (192.168.1.31, 14GB, 128GB disk) — all workloads run on the one worker
- **Longhorn runs 1 replica** (single-replica volumes, CSI sidecars at 1 replica) — node removal requires replica eviction first or data is lost
- **Worker has AMD iGPU passthrough** (node label `gpu: amd`) — jellyfin's nodeSelector depends on it
- **No metrics-server** — `kubectl top` doesn't work; query VictoriaMetrics (vmsingle, port 8428) instead
- **Talos Linux** — no systemd, no SSH; kube-scheduler/controller-manager/etcd/kube-proxy metrics not exposed; Cilium replaces kube-proxy
- **Gateway API** — used instead of Ingress everywhere; external-dns sources are `gateway-httproute`, `gateway-grpcroute`, `gateway-tlsroute`
- **Cilium L2 announcements** — `l2announcements.enabled: true` + `externalIPs.enabled: true` required; Proxmox VMs use `ens*` interfaces (not `eth*`/`enp*`)
- **Cilium must set `cluster.name: homelab`** — omitting this breaks Hubble relay TLS
- **VM Operator webhooks must stay disabled** — `admissionWebhooks.enabled: false` to avoid race condition
- **Cilium operator: 1 replica** — 2 replicas cause restarts on the resource-constrained CP node

## Monitoring Architecture

Victoria-only stack with fan-out collectors:
```
VMAgent ──remote-write──→ VMSingle (10Gi)
Vector DaemonSet ────────→ VictoriaLogs (5Gi)
OTel Collector ──────────→ VictoriaTraces (5Gi)
```
Grafana-ecosystem backends (Prometheus, Loki, Tempo) were removed to save resources. Grafana at grafana.corruptmane.xyz with Zitadel OIDC, dashboards provisioned via sidecar (vm-k8s-stack + Cilium ConfigMaps) and Grafana.com (Longhorn, CNPG, K8S Dashboard).

## SSO Architecture

Zitadel at auth.corruptmane.xyz — local user management, offline-capable, Google account linking planned. Grafana authenticates via OIDC (generic_oauth). Zitadel login UI is a separate service (`zitadel-login:3000`) requiring its own HTTPRoute rule for `/ui/v2/login`.
