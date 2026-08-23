# RCA: Zitadel outage after power loss (2026-08-11)

**Status:** RESOLVED — service recovered 2026-08-11 ~12:07 UTC
**Date:** 2026-08-11
**Affected service:** Zitadel (`auth.corruptmane.xyz`) + its CNPG PostgreSQL (`zitadel-db`)
**Blast radius:** Zitadel SSO + `zitadel-login` UI. Everything else in the cluster recovered.
**Author:** homelab debugging session

---

## 1. Executive summary

After an unscheduled power loss, the entire Kubernetes cluster recovered **except Zitadel**. All five Zitadel-related pods have been crash-looping since reboot (110–150 restarts, 12h uptime).

The root cause is a **single corrupted Postgres metadata file** — `pg_logical/replorigin_checkpoint` inside the CNPG data volume. The file's 4-byte magic header was zeroed by a torn (partial) write during the power cut. On every startup, Postgres reads this file, detects `magic 0` instead of the expected `307747550`, and immediately PANICs (`PANIC: replication checkpoint has wrong magic 0 instead of 307747550`) before completing crash recovery.

The database data itself is intact (`pg_controldata` shows a valid checkpoint, and the PANIC is not a WAL corruption). The fix applied is the well-documented one: **rename the corrupt file aside; Postgres regenerates it on the next checkpoint** — performed via the `kubectl cnpg` plugin's *fencing* mode, which keeps the crash-looping instance alive for maintenance without any extra pod (see §9.1).

A significant secondary finding: **this CNPG cluster has no S3/Barman backup stanza configured**, so there is no restore-from-backup fallback today. This is a resilience gap (see §11).

---

## 2. Symptom / user impact

- `auth.corruptmane.xyz` (Zitadel) unreachable.
- Grafana OIDC login (which relies on Zitadel) degraded.
- Everything else in the cluster (`flux get all` / pod sweep) recovered normally.
- **Resolved:** all services back up ~12:07 UTC on 2026-08-11 (see §9.1).

---

## 3. Timeline

| Time (UTC) | Event |
|---|---|
| 2026-08-10 18:30:48 | Last clean Postgres checkpoint (`pg_controldata` "last modified"). |
| 2026-08-10 ~18:30 | Power outage hits the Proxmox host. |
| 2026-08-10 ~18:31 | Postgres killed mid-write; `replorigin_checkpoint` header write torn/zeroed. |
| 2026-08-10 ~21:11 | Nodes back, cluster reconciled by Flux. CNPG reports `ConsistentSystemID=False`, `Ready=False` (from cluster status conditions). |
| 2026-08-10 → 11 | All 5 Zitadel pods crash-loop through the night (110–150 restarts). No alerting fired. |
| 2026-08-11 ~11:30 | Investigation begins. |
| 2026-08-11 11:5x | RCA written to `docs/zitadel-poweroutage-rca-2026-08-11.md`. |
| 2026-08-11 ~12:00 | `cnpg` kubectl plugin installed; instance fenced, corrupt file renamed aside. |
| 2026-08-11 ~12:07 | Instance unfenced → postgres recovers, regenerates `replorigin_checkpoint`; Zitadel + login pods healthy (0 restarts). CNPG `Ready=True`. |

---

## 4. Environment context

- Two-node Talos Linux cluster: `talos-30r-dp1` (control-plane) + `talos-z9j-v4k` (worker). Both `Ready`.
- All workloads run on the single worker `talos-z9j-v4k`.
- Storage: **Longhorn, 1 replica** on the worker (single-replica volume).
- Zitadel database: CNPG `Cluster zitadel-db`, 1 instance, 2Gi PVC `zitadel-db-1`, `storageClass: longhorn`.
- DB settings of note: `wal_level: logical`, `archive_mode: on`, `full_page_writes: on`.
- Flux HelmRelease `zitadel` (chart `zitadel@9.26.0`) — status `True`.
- No CNPG `backup:` stanza — **no S3/Barman backups** configured on this cluster.

---

## 5. Investigation log

Every step below is reproduced as executed. This is the teachable part — note how the *first three steps are cheap, read-only, and narrow the space fast.*

### Step 1 — Broad sweep: pods, HelmReleases, nodes (parallel)

```bash
kubectl get pods -A | grep -i zitadel
kubectl get helmreleases -A | grep -i zitadel
kubectl get nodes -o wide
```

Evidence:

```
zitadel/zitadel-59d64d7b6d-8vjd6    0/1  Init:0/1     111  12h
zitadel/zitadel-8495676cfb-c9jmh    0/1  Init:0/1     111  12h
zitadel/zitadel-db-1                0/1  CrashLoopBackOff  149  12h
zitadel/zitadel-login-6895d9989c-9pvn4  0/1  Init:Error  111  12h
zitadel/zitadel-login-8578c5679c-wlspn  0/1  Init:Error  110  12h

NAME            STATUS  ROLES  VERSION    INTERNAL-IP    ...
talos-30r-dp1   Ready   control-plane  v1.35.2  192.168.1.30
talos-z9j-v4k   Ready   <none>         v1.35.2  192.168.1.31

helmrelease zitadel  149d  True  Helm install succeeded (chart zitadel@9.26.0)
```

Takeaways:

- **Nodes are healthy** — not a node problem.
- **HelmRelease `True` is a trap.** Flux's HelmRelease only tracks the Helm release object, **not** pod health. A crash-looping app still shows `True`. You cannot rely on it for health — this is by design; the real health signal is pod readiness/events (and ideally an alert).
- 111–150 restarts over 12h ⇒ crash-looping since the reboot, not a fresh breakage.
- Both the app and its DB are affected — the DB pod (`zitadel-db-1`) is the deepest component in the dependency chain. Start there.

### Step 2 — Pod detail + events (parallel)

```bash
kubectl -n zitadel get pods -o wide
kubectl -n zitadel get events --sort-by=.lastTimestamp | tail -40
```

Evidence:

```
zitadel-db-1  0/1  CrashLoopBackOff  149 ...  NODE: talos-z9j-v4k

Events:
Warning BackOff pod/zitadel-8495676cfb-c9jmh  Back-off restarting failed container wait-for-postgres
Warning BackOff pod/zitadel-login-8578c5679c-wlspn  Back-off restarting failed container wait-for-zitadel
Warning BackOff pod/zitadel-db-1  Back-off restarting failed container postgres
```

Takeaways:

- The events reveal a **dependency chain**:

```
wait-for-postgres (fails) → waits on → postgres (CrashLoopBackOff)
wait-for-zitadel  (fails) → waits on → zitadel main pod (which waits on postgres)
```

- The `wait-for-postgres` / `wait-for-zitadel` init containers (`wait4x`) are **symptoms, not the cause** — they fail simply because their dependency never became ready. The actual broken component is **postgres**.

**Debugging lesson:** always chase the *deepest* failing container in the dependency graph, not the first red `Init` you see.

### Step 3 — Read the crashing container's logs (with `--previous`)

```bash
kubectl -n zitadel logs zitadel-db-1 -c postgres --previous --tail=60
```

Evidence (trimmed):

```
pg_controldata: Database cluster state: in production
pg_controldata: Latest checkpoint location: A6/1A001C60 ...  valid checkpoint
LOG:  database system was interrupted; last known up at 2026-08-10 18:30:49 UTC
LOG:  database system was not properly shut down; automatic recovery in progress
PANIC: replication checkpoint has wrong magic 0 instead of 307747550
LOG:  startup process (PID 30) was terminated by signal 6: Aborted
LOG:  aborting startup due to startup process failure
LOG:  database system is shut down
```

Takeaways:

- `--previous` is essential when the container is in CrashLoopBackOff — the *current* container's logs may be empty or show only the latest crash.
- The **checkpoint is valid** (`pg_controldata` parses fine, state `in production`) ⇒ WAL / data files are most likely intact.
- Postgres detected the unclean shutdown (expected after a power cut) and started automatic crash recovery, then **PANIC'd within the first second**. Note the PANIC happens *before* WAL replay completes — it is not a WAL-corruption panic.

### Step 4 — Confirm CNPG / volume state (parallel)

```bash
kubectl -n zitadel get cluster,backup,volume,pvc -o name
kubectl -n zitadel get cluster zitadel-db -o jsonpath='{.status}'   # trimmed
kubectl -n zitadel get cluster zitadel-db -o yaml | sed -n '/^spec:/,/^status:/p'
kubectl -n zitadel get pod zitadel-db-1 -o jsonpath='{...volumeMounts...}'
```

Evidence:

- PVC `zitadel-db-1` → **Bound**, 2Gi, `longhorn`.
- Pod mounts `pgdata` PVC at `/var/lib/postgresql/data` (⇒ PGDATA = `/var/lib/postgresql/data/pgdata`).
- Cluster spec: `instances: 1`, `wal_level: logical`, **no `backup:` stanza** (no S3/Barman destination).
- Cluster conditions: `ConsistentSystemID=False` ("no instance reported a system ID"), `Ready=False`, `ContinuousArchiving=True`.

Takeaways:

- PGDATA lives on the PVC; any file surgery must go through the volume.
- `wal_level: logical` is the reason the `replorigin_checkpoint` file exists and is rewritten at every checkpoint — Zitadel uses logical decoding features, and this metadata file tracks replication-origin progress.
- **No backup stanza** = no restore-from-backup escape hatch (important gap, §11).

### Step 5 — Research the exact PANIC string (web)

Search: `PostgreSQL 17 panic "replication checkpoint has wrong magic" power loss recovery`

Findings (multiple independent sources, PG 11–16, all identical signature after power loss):

- PostgreSQL mailing list (2020): corruption of `$PGDATA/pg_logical/replorigin_checkpoint`; removal fixes it.
- Broadcom KB, HPE Ezmeral KB: same error ⇒ remove `replorigin_checkpoint`.
- Multiple homelab reports (TeslaMate, Harbor, etc.): rename/remove `replorigin_checkpoint` → Postgres regenerates it.
- Postgres source (`src/backend/replication/logical/origin.c`, `StartupReplicationOrigin()`): reads the file, validates the first 4 bytes against `REPLICATION_STATE_MAGIC`, PANICs on mismatch.

---

## 6. Root cause analysis

### 6.1 The failure chain

```
Power cut → postgres hard-killed mid-write
  → pg_logical/replorigin_checkpoint header write torn (magic → 0)
      → postgres startup: StartupReplicationOrigin() reads file, magic mismatch
          → PANIC: "replication checkpoint has wrong magic 0 instead of 307747550"
              → postgres exits → CrashLoopBackOff (149 restarts)
                  → wait-for-postgres init fails (zitadel, zitadel-login pods)
                      → entire Zitadel stack down
```

### 6.2 Why this specific file, and why it's so fragile

- `replorigin_checkpoint` is a tiny file (magic + up to N `ReplicationStateOnDisk` structs + CRC) that Postgres **rewrites at every checkpoint** (`CheckPointReplicationOrigin`). With `wal_level: logical`, `max_active_replication_origins > 0`, so the file is always written.
- On startup, `StartupReplicationOrigin()` reads the first 4 bytes and compares to `REPLICATION_STATE_MAGIC = 307747550`. A torn write that zeroes or truncates those 4 bytes ⇒ instant PANIC — before any WAL recovery can proceed.
- Torn writes happen when power is cut while the storage layer has un-flushed data. With **Longhorn at 1 replica on a single node**, there is no redundancy and no other copy of the last writes — the volume's last sector(s) are whatever the kernel happened to flush.

### 6.3 What was *not* corrupted

- `pg_controldata` reads cleanly; checkpoint is valid; `pg_control` timestamps match the outage window.
- The PANIC message is specific to the origin file, *not* WAL. A WAL corruption would produce a different signature (`invalid record length`, `could not read from WAL file`, etc.).
- Conclusion: **the Zitadel data itself is almost certainly safe.** Worst case (if WAL replay later fails) is losing a few minutes of transactions ending at the last checkpoint — not a full restore.

### 6.4 Why the diagnosis was initially confusing

| Misleading signal | Why it misled | Resolution |
|---|---|---|
| HelmRelease `True` | Tracks Helm release state, not pod health | Check pods/events for real health |
| `Init:0/1` on app pods | init failure looks like the app's problem | Trace the dependency: init waits on DB |
| `wait-for-postgres` failing | looks like a networking issue | It's just DB never became ready |
| Postgres "automatic recovery in progress" | sounds like it will self-heal | It PANIC'd immediately after that line |

---

## 7. Impact assessment

- **Data safety:** High confidence data is intact (valid checkpoint, no WAL-corruption signature). Zitadel users and config should survive.
- **Recovery options ranked:**
  1. Remove/rename `replorigin_checkpoint` (primary fix — documented, non-destructive, reversible).
  2. `pg_resetwal -f` as last resort — would discard WAL since last checkpoint (loses recent transactions), only if replay later hits WAL corruption. Not expected to be needed.
  3. Restore from S3/Barman backup — **NOT AVAILABLE** (no backup stanza). Gap to fix (§11).
- **Risk of the primary fix:** essentially zero. If postgres regenerates the file on next checkpoint (it does), Zitadel comes up. The rename preserves the corrupt file for forensics.

---

## 8. Constraints discovered (why the fix isn't a one-liner)

- The crash-looping pod's container exits ~1s after start ⇒ `kubectl exec` is a race, not viable.
- The `cnpg` kubectl plugin was **not installed at investigation time** (installed during the fix — see §9.1).
- The file lives on the PVC ⇒ any direct fix must go through the mounted volume.
- The PVC is RWO/Longhorn attached on `talos-z9j-v4k` ⇒ an adhoc maintenance pod would have to run on the **same node** (same-node RWO mounts are allowed).

---

## 9. Fix options considered

Two viable paths were identified:

- **Option A — adhoc maintenance pod** (proposed first, *not used*): mount the PVC from a throwaway pod on the same node and rename the file there. Works without any plugin, but adds a pod, a manual mount, and a node-affinity requirement.
- **Option B — CNPG fencing via `kubectl cnpg` plugin** (*used — preferred*): fence the instance so the instance-manager keeps the pod alive without starting postgres, then `exec` in directly. No extra pod, no PVC mounting, operator-aware and reversible. This is CNPG's official maintenance mechanism.

### 9.1 Resolution — executed 2026-08-11 (~12:00–12:07 UTC)

Option B was performed manually. The exact flow:

```bash
# 0. Install plugin (done by operator)
kubectl krew install cnpg   # or other install method

# 1. Stop the crash-loop: instance-manager keeps pod Running, does NOT start postgres
kubectl cnpg fencing on zitadel-db zitadel-db-1 -n zitadel

# 2. Inspect the corrupt file (first 4 bytes zeroed = magic 0 instead of 307747550)
kubectl -n zitadel exec zitadel-db-1 -- bash -c \
  'ls -la /var/lib/postgresql/data/pgdata/pg_logical/ && \
   od -A x -t x1z /var/lib/postgresql/data/pgdata/pg_logical/replorigin_checkpoint | head'

# 3. Rename it aside (reversible; kept for forensics)
kubectl -n zitadel exec zitadel-db-1 -- bash -c \
  'mv /var/lib/postgresql/data/pgdata/pg_logical/replorigin_checkpoint \
     /var/lib/postgresql/data/pgdata/pg_logical/replorigin_checkpoint.bak-20260811'

# 4. Bring it back: postgres starts, WAL recovery completes, file regenerates
kubectl cnpg fencing off zitadel-db zitadel-db-1 -n zitadel

# 5. Watch the cascade self-heal
kubectl -n zitadel get pods -w
```

**Why fencing makes this easy (vs. the adhoc pod):**

| | Option A: adhoc pod | Option B: CNPG fencing |
|---|---|---|
| Extra pod | Yes, mounts PVC manually | No |
| PVC concurrency | Must match node, RWO mount dance | None — files already mounted in the pod |
| Safety | Operator unaware of maintenance | Operator holds the instance (won't delete/recreate it) |
| Reversibility | Manual pod deletion | `kubectl cnpg fencing off` |

**Confirmed post-recovery state** (evidence captured 2026-08-11 12:07 UTC):

```bash
$ kubectl -n zitadel get pods
zitadel-59d64d7b6d-8vjd6       1/1 Running  0  13h
zitadel-db-1                   1/1 Running  155 (11m ago)  13h
zitadel-login-8578c5679c-wlspn 1/1 Running  0  13h
# restarts frozen (155 = pre-fix counter); old ReplicaSets cleaned up

$ kubectl -n zitadel get cluster zitadel-db \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} ({.reason}){"\n"}{end}'
ConsistentSystemID=True (Unique)
Ready=True (ClusterIsReady)
ContinuousArchiving=True (ContinuousArchivingSuccess)

$ kubectl -n zitadel exec zitadel-db-1 -- ls -la /var/lib/postgresql/data/pgdata/pg_logical/
replorigin_checkpoint                 8 bytes  Aug 11 12:07   # regenerated
replorigin_checkpoint.bak-20260811    8 bytes  Aug 10 18:35   # corrupt original, preserved
```

**Side observation (not blocking):** immediately after recovery, postgres logged several
`FATAL 53300 "remaining connection slots are reserved for roles with the SUPERUSER attribute"`
during Zitadel's startup connection burst — `max_connections` is only `20` and all slots were
taken at once. Zitadel retried and settled into Running/healthy, so this was transient, but the
connection ceiling is tight and worth revisiting if Zitadel ever reports connection errors.

---

## 10. Verification (post-fix)

The checklist below was executed after recovery — **all items passed**, evidence in §9.1.

```bash
# 1. DB comes back
kubectl -n zitadel get pods -w
kubectl -n zitadel logs zitadel-db-1 --tail=30          # look for "ready to accept connections"

# 2. Zitadel fully up
kubectl -n zitadel get pods                              # all Ready, restarts reset

# 3. Postgres recovered state
kubectl -n zitadel get cluster zitadel-db \
  -o jsonpath='{.status.conditions[*].reason}'           # ConsistentSystemID / Ready True

# 4. New file regenerated
kubectl -n zitadel exec zitadel-db-1 -- sh -c 'ls -la /var/lib/postgresql/data/pgdata/pg_logical/'

# 5. Service responds end-to-end
curl -k -s -o /dev/null -w '%{http_code}\n' https://auth.corruptmane.xyz/healthz

# 6. Confirm restarts stopped climbing
kubectl -n zitadel get pods                             # RESTARTS column stable

# 7. Reconcile Flux (optional, ensures git == cluster)
flux reconcile kustomization flux-system --with-source
```

---

## 11. Prevention & hardening recommendations (follow-up, not blocking the fix)

1. **Add CNPG S3 backups.** The cluster has *no* `backup:` stanza despite the SSM params (`/homelab/cnpg-backup-*`) already existing in AWS. Add a `backup:`/`barmanObjectStore` block so this recovery path exists. Verify by creating a `ScheduledBackup` and testing `kubectl cnpg backup` / restore into a throwaway cluster.
2. **Crash-consistency of Longhorn (1 replica).** The torn write is a storage-durability failure. Options:
   - Add a second Longhorn replica (needs a second node with storage — currently single host).
   - At minimum, acknowledge that single-replica Longhorn volumes are *not* crash-durable for the last writes; keep all stateful apps' backups fresh.
3. **Alert on CrashLoopBackOff.** This sat silently for 12h. Add a VictoriaMetrics/Grafana alert (e.g. on `kube_pod_container_status_restarts_total > 5` or a `kube-state-metrics` crash-loop query) covering `KubernetesDeployment`-level health — not just Flux/HelmRelease status, which stays green.
4. **`wal_level: logical`** on a single-instance DB is only needed if Zitadel truly uses logical decoding. If not, dropping it also removes the `replorigin_checkpoint` exposure class. (Verify Zitadel requirements before changing.)
5. **Done — `cnpg` plugin now installed** (v1.30.0, installed 2026-08-11). Future maintenance can use `kubectl cnpg fencing on/off` (used for this fix, see §9.1) instead of a manual PVC-mounting pod.
6. **Add a UPS / shutdown hook** for the Proxmox host so postgres gets a clean shutdown on future outages (best mitigation for a single-node storage layer).

---

## 12. References

- PostgreSQL mailing list — PANIC: replication checkpoint has wrong magic 0 instead of …
  https://www.postgresql.org/message-id/CA%2BXUKciJpR-Wxt2irzjToYisMMPnoZA7Mc3h-CUf56JtJEsvAQ%40mail.gmail.com
- David Chua (SRE) — Fixing Postgresql Replication Checkpoint Wrong Magic 0 Error
  https://dchua.com/posts/2024-08-01-fixing-postgresql-replication-checkpoint-wrong-magic-0-error/
- Broadcom KB — EDR: Postgres service failed to start due to PANIC replication checkpoint has wrong magic 0
  https://knowledge.broadcom.com/external/article/286192/
- HPE Ezmeral — keycloak-postgres PANIC workaround (remove `replorigin_checkpoint` from data volume)
  https://docs.ezmeral.hpe.com/unified-analytics/15/Troubleshooting/ts-user-interface.html
- PostgreSQL source — `src/backend/replication/logical/origin.c` (`StartupReplicationOrigin`, `CheckPointReplicationOrigin`)
  https://github.com/postgres/postgres/blob/master/src/backend/replication/logical/origin.c
- AskUbuntu — Postgres 12 "replication checkpoint has wrong magic" (ownership + file removal fix)
  https://askubuntu.com/questions/1311254/

---

*Investigation performed 2026-08-11. All commands run against the homelab cluster; evidence captured verbatim above.*
