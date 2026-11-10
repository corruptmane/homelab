# cvgen hibernation

cvgen is parked: its GitRepository and all eight Flux Kustomizations have
`spec.suspend: true`. The live application deployments, NATS StatefulSet, and
Flagger controller are scaled to zero. PostgreSQL has
`cnpg.io/hibernation: "on"`. The synthetic CronJob, CNPG ScheduledBackup,
HelmReleases, and image automation are also suspended.

Keep the namespaces, HelmReleases, CNPG Cluster, PVCs, and PVs. Removing cvgen
from `apps/kustomization.yaml` can trigger pruning; suspension is intentional.

## Monitoring cleanup while parked

Stopping pods does not remove their disruption budgets or scrape definitions.
Two leftover resources caused alerts after shutdown:

| Alert | Resource | Reason |
| --- | --- | --- |
| `KubePdbNotEnoughHealthyPods` | PDB `cvgen/cvgen-db-primary` | Requires one healthy primary, but PostgreSQL is hibernated. |
| `ScrapePoolHasNoTargets` | VMPodScrape `flagger-system/flagger` | Flagger is scaled to zero, so discovery finds no pods. |

After confirming PostgreSQL reports `Hibernated`, Flagger is at zero replicas,
and both `cvgen-db` and `cvgen-infra` are suspended, disable the database PDB
and remove the Flagger scrape:

```sh
kubectl --context admin@homelab patch clusters.postgresql.cnpg.io cvgen-db -n cvgen \
  --type=merge -p '{"spec":{"enablePDB":false}}'
kubectl --context admin@homelab delete vmpodscrapes.operator.victoriametrics.com flagger -n flagger-system
```

The installed CNPG 1.28.1 controller recreates a deleted PDB even while this
cluster is hibernated (verified during cleanup). Use the supported
`Cluster.spec.enablePDB` setting so the operator removes and stops recreating
it. This is a temporary live override while the database Kustomization is
suspended; **explicitly set it back to `true` before restarting PostgreSQL**.
See the [CNPG API reference](https://cloudnative-pg.io/docs/devel/cloudnative-pg.v1/).

The Flagger scrape manifest remains in the application repository at
`deploy/k8s/infra/flagger-scrape.yaml`; the suspended `cvgen-infra`
Kustomization will not restore it until resumed. NATS's existing PDB has zero
desired healthy pods at zero replicas and does not need removal.

No fleet alert rules or notification routes need to be disabled. Allow time
for scrape configuration reloads, metric staleness, and alert evaluation after
cleanup; an already delivered notification remains in chat history.

## Verification and restoration

Verify there are no running cvgen pods and both existing PVCs remain `Bound`:

```sh
kubectl get pods,pvc -n cvgen
kubectl get pdb -n cvgen
kubectl get vmpodscrapes.operator.victoriametrics.com -n flagger-system
```

When bringing cvgen back, reverse the Git suspension flags and the live
shutdown settings. Restore NATS and PostgreSQL before the application pods;
restore the synthetic job and backups after their dependencies are ready.
Explicitly restore Helm-managed replica counts as needed: resuming a
HelmRelease alone may not undo manual scaling without an upgrade.

For the monitoring resources removed above:

- Restore PDB management before turning off hibernation:

  ```sh
  kubectl --context admin@homelab patch clusters.postgresql.cnpg.io cvgen-db -n cvgen \
    --type=merge -p '{"spec":{"enablePDB":true}}'
  kubectl --context admin@homelab annotate clusters.postgresql.cnpg.io cvgen-db -n cvgen \
    cnpg.io/hibernation=off --overwrite
  ```

  Verify that `cvgen-db-primary` exists after the database is ready. Do not
  rely on resuming Flux alone to reset the field: the source manifest omits it.
- Restore Flagger's deployment and reconcile `cvgen-infra` after resuming its
  source and Kustomization. Flux will recreate the Flagger `VMPodScrape`.
- Verify the Flagger scrape has a healthy target and the PDB's healthy pod
  count meets its desired count.
