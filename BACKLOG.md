# Backlog

## Fleet monitoring exporters / HPA enablement (Task 4, future)

For cvgen saturation alerts and any future HPA-on-consumer-lag the fleet lacks:

- prometheus-nats-exporter (cvgen NATS consumer lag)
- postgres_exporter (CNPG metrics beyond operator defaults)
- metrics-server (`kubectl top`; note: none today — query VictoriaMetrics vmsingle instead)
- prometheus-adapter (custom-metrics HPA)

If/when wanted: propose installs following the controllers-layer HelmRelease pattern
(`infrastructure/controllers/<name>/`, pinned semver, PSA labels where privileged).
Do not install until there is a concrete consumer.

## Flux substitution escape hatch

`infrastructure-configs` now carries a non-optional `postBuild.substituteFrom`
(Secret `zitadel-db-destination`). While `/homelab/cnpg-backup-bucket-name` is absent
in SSM the whole configs layer pauses reconciliation (fail-loud by design; applied
resources are untouched because substitution failure happens pre-apply). If the
configs layer ever hosts multi-project substitutions or the pause becomes a stall
risk, extract a path-scoped child Kustomization
(`path: ./infrastructure/configs/zitadel`, own substituteFrom,
`dependsOn: infrastructure-controllers`).

Note: kustomize-controller resolves `substituteFrom` sources only inside the
Kustomization's own namespace (verified against kustomize-controller v1.8.1 /
fluxcd/pkg/kustomize v1.27.0) — any future substitution Secret must be rendered
into `flux-system`.

## cvgen release-train coupling

Because the live chain is `cvgen-db -> migrations -> apps -> canary -> alerts`,
a failed `cvgen-db` (e.g. substitution secret missing) freezes reconciliation of ALL
downstream cvgen Kustomizations. Running workloads are unaffected, but new cv-repo
changes stop rolling out until the SSM parameter exists and reconciliation heals.
