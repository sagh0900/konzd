# Zabbix Operator — Production Failover Runbook

**Version**: v2.41
**Tested**: 2026-02-28 (scenarios 1–9) + 2026-03-07 (CNPG PG15→16 + Zabbix 7.0.21→7.4 upgrade)
**Cluster**: 7-node (ccs-test-master1/2/3 + ccs-test-worker1/2/3/4)
**Zabbix**: 7.4.0 in full HA mode (decentralized / link mode)
**CNPG**: 2-node cluster (zabbix-pg-1/2), PostgreSQL 16
**Operator images**: `zabbix-operator:2.41`, `zabbix-lease-sidecar:2.1`, `zabbix-migration-sentinel:2.3`

---

## 1. HA Architecture

```
lease-sidecar (2s)       supervisor (1s poll)        controller (Lease watch)
  |                           |                              |
  +-- writes heartbeat  -->   +-- checks age > 8.5s  -->    +-- Lease event fires immediately
  |   /var/zabbix-heartbeat   |   if stale: SIGKILL          |   resolves leader pod
  |   /beat (Unix nanos)      |   child zabbix_server         |   if Ready: update EPS
  |                           |                              |   else: clear EPS + requeue 2s
  +-- renews K8s Lease  -->
      every ~6.67s
```

### Lease Parameters
| Parameter | Value | Effect |
|-----------|-------|--------|
| leaseDuration | 20s | Max time before standby steals lease |
| renewDeadline | 15s | Leader stops if it can't renew within 15s |
| retryPeriod | 3s | Standby retry interval for acquisition |
| heartbeatInterval | 2s | Sidecar writes heartbeat file every 2s |
| T_hb threshold | 8.5s | Supervisor kills zabbix_server if heartbeat stale |

### Lease-Sidecar Active Detection (v2.1)

The lease-sidecar (v2.1+) probes `127.0.0.1:10051` to detect Zabbix HA mode before acquiring
the K8s Lease. Only the Zabbix HA active leader listens on port 10051 — standby nodes do not.

```
Without v2.1:  K8s Lease winner = first pod to start = may be Zabbix STANDBY
               → EPS points to standby pod → port 10051 refused → "server not reachable"

With v2.1:     sidecar probes :10051 before acquiring → only ACTIVE pod holds K8s Lease
               → EPS always points to Zabbix HA active leader → port 10051 open ✓
```

On Zabbix HA failover (old active → standby), the sidecar detects `:10051` no longer listening,
voluntarily releases the K8s Lease (without fencing — standby process stays alive), and the
new active pod's sidecar acquires within one `retryPeriod` (~3s).

### ha_node.address and ZBX_SERVER_HOST (v2.24+)

The configure-db init container writes `NodeAddress=<server-name>-active:10051` to
`zabbix_server.conf`. This causes Zabbix to register in the `ha_node` table with:
```
ha_node.address = "my-zabbix-server-active"
```

The web controller sets `ZBX_SERVER_HOST = "my-zabbix-server-active"` (DNS hostname, not ClusterIP).
PHP `isRunning()` performs a string comparison: `ha_node.address == ZBX_SERVER_HOST` → must match.
`DB_SERVER_HOST` (PgBouncer) uses ClusterIP to bypass UDP DNS issues at pod startup.

### EndpointSlice Routing
- `my-zabbix-server-active` — selector-less Service with operator-managed EndpointSlice
- Operator watches K8s Lease; resolves leader pod IP only if Running + all containers Ready
- **DB gate (v2.7)**: EPS withheld if `ZabbixDatabase.status.conditions[DatabaseReady]` is False
- Idempotent: existing IP compared before writing; RetryOnConflict used
- **Lease sync (v2.1)**: K8s Lease holder is always the Zabbix HA active node (`:10051` probe)

### Key Service Names
- Leader traffic: `my-zabbix-server-active` (EndpointSlice-managed, selector-less)
- Agent to Server: `my-zabbix-server-active` (v2.9 fix — was incorrectly bare CR name)
- DB pooler: `zabbix-pg-pooler-rw` (ClusterIP used for DB_SERVER_HOST in web pods)
- DB direct (DDL): `zabbix-pg-rw` (ClusterIP used in configure-db init container)
- Agent passive: `my-zabbix-agent` (created only when mode=passive or mode=both)

---

## 2. Pre-Test Monitoring Setup

```bash
# Terminal A — EPS + Lease (1s refresh)
watch -n1 'echo "=== EPS ===" && kubectl get endpointslices my-zabbix-server-active \
  -n zabbix-suite -o jsonpath="{.endpoints[*].addresses}" 2>/dev/null; \
  echo; echo "=== Lease ===" && kubectl get lease \
  zabbix-server-leader-my-zabbix-server -n zabbix-suite \
  -o jsonpath="holder={.spec.holderIdentity} trans={.spec.leaseTransitions}" 2>/dev/null'

# Terminal B — operator logs (failover-relevant lines)
kubectl logs -n zabbix-operator -l app.kubernetes.io/name=zabbix-operator \
  -f --prefix 2>/dev/null | grep -E "EndpointSlice|leader|Leader|Lease|reconcil"

# Terminal C — server pods watcher
kubectl get pods -n zabbix-suite -l app.kubernetes.io/name=zabbix-server -w

# Terminal D — continuous HTTP probe
while true; do
  CODE=$(curl -sk -o /dev/null -w "%{http_code}" --max-time 3 \
    https://zabbix.ccs-test.ccs.shibuya.se/index.php)
  printf "$(date +%H:%M:%S)  HTTP=%s\n" "$CODE"; sleep 2
done
```

---

## 3. Post-Test Health Verification

```bash
echo "=== HEALTH CHECK ==="
kubectl get pods -n zabbix-suite -o wide | grep -v agent
kubectl get zabbixdatabase my-zabbix-db -n zabbix-suite
kubectl get endpointslices my-zabbix-server-active -n zabbix-suite \
  -o jsonpath='eps={.endpoints[0].addresses[0]}'
echo ""
kubectl get lease zabbix-server-leader-my-zabbix-server -n zabbix-suite \
  -o jsonpath='holder={.spec.holderIdentity} trans={.spec.leaseTransitions}'
echo ""
kubectl get cluster zabbix-pg -n zabbix-suite \
  -o jsonpath='primary={.status.currentPrimary} ready={.status.readyInstances}'
echo ""
curl -sk -o /dev/null -w "HTTP=%{http_code}" \
  https://zabbix.ccs-test.ccs.shibuya.se/index.php
```

**Healthy state**: ZabbixDatabase READY=True, ZabbixServer READY=True, EPS has 1 endpoint
matching leader pod IP, CNPG ready=3, HTTP=200.

---

## 4. Failover Scenario Test Results

All 9 scenarios were executed on 2026-02-28 against the production cluster.

### Baseline at Test Start
- Leader pod: `my-zabbix-server-6dd86b7c77-92vfh` on `ccs-test-worker1` (10.244.3.18)
- Standby pod: `my-zabbix-server-6dd86b7c77-gtghf` on `ccs-test-worker3`
- CNPG primary: `zabbix-pg-1` on `ccs-test-worker1`
- Operator: 2 pods on worker3 + worker4
- Lease transitions at start: 4

---

### Scenario 1 — Graceful Leader Pod Deletion (SIGTERM)

**Failure type**: Graceful pod delete via `kubectl delete pod`
**Expected RTO**: 5-10s | **Actual RTO**: ~1s | **HTTP gap**: 0s | **Result**: PASS

**Theory**: SIGTERM triggers the lease-sidecar's deferred cleanup, releasing the Lease before
the pod exits. Standby acquires in the next retryPeriod tick.

**Commands**:
```bash
LEADER=$(kubectl get lease zabbix-server-leader-my-zabbix-server \
  -n zabbix-suite -o jsonpath='{.spec.holderIdentity}')
kubectl delete pod "$LEADER" -n zabbix-suite
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | SIGTERM sent; lease-sidecar releases Lease immediately |
| T+1s | Standby acquires Lease; operator updates EPS to standby IP |
| T+1s | HTTP 200 continuous — no gap |

**Lease transitions**: 4 → 5

**Key finding**: Graceful release by the sidecar before pod exit is highly effective. Standby
acquires within one retryPeriod tick (~1s observed vs 3s max). This is the fastest possible
failover path.

---

### Scenario 2 — Force-Kill Leader Pod (SIGKILL)

**Failure type**: `kubectl delete pod --force --grace-period=0`
**Expected RTO**: 20-25s | **Actual RTO**: ~2s | **HTTP gap**: 0s | **Result**: PASS

**Theory**: Force-delete bypasses SIGTERM — no graceful Lease release. Expected to wait full
leaseDuration (20s) TTL. Actual: standby detects pod gone from API server and acquires early.

**Commands**:
```bash
LEADER=$(kubectl get lease zabbix-server-leader-my-zabbix-server \
  -n zabbix-suite -o jsonpath='{.spec.holderIdentity}')
kubectl delete pod "$LEADER" -n zabbix-suite --force --grace-period=0
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | Pod force-deleted; API removes pod object immediately |
| T+2s | Standby detects holder pod gone from API; acquires Lease |
| T+2s | Operator updates EPS to standby IP |

**Lease transitions**: 5 → 6

**Key finding**: Force-kill RTO is ~2s, NOT 20s. When Kubernetes force-deletes a pod on a
managed cluster, the pod object is removed from the API immediately. The standby's lease-election
code detects the holder pod is gone and acquires without waiting for the full TTL.
IMPORTANT: A true node crash (node unreachable, not pod deleted) would still require the full
20s leaseDuration TTL before the standby can acquire.

---

### Scenario 3 — Kill Both Server Pods Simultaneously

**Failure type**: Both pods SIGKILL'd at once; no standby available
**Expected RTO**: 45-60s | **Actual RTO**: ~18s (pods ready) | **HTTP gap**: 0s | **Result**: PASS

**Commands**:
```bash
kubectl delete pods -n zabbix-suite -l app.kubernetes.io/name=zabbix-server \
  --force --grace-period=0
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | Both pods SIGKILL'd |
| T+0s | EPS not immediately cleared (operator reconcile lag up to 60s) |
| T+18s | New pods reach 2/2 Ready; first pod acquires Lease (trans 6 -> 7) |
| T+~30s | Operator reconcile fires; EPS updated to new leader IP |
| T+0s-T+30s | HTTP 200 continuous |

**Key finding**: EPS was not cleared immediately because the operator's reconcile loop runs on a
60s period (triggered also by Lease watch events). HTTP stayed 200 because ZabbixWeb UI serves
from its own DB connection independently of the ZabbixServer EPS.

---

### Scenario 4 — Worker Node Drain (ccs-test-worker1)

**Failure type**: Full node drain — hosting leader pod AND CNPG primary simultaneously
**Expected RTO**: 30-50s | **Actual RTO**: CNPG T+1s, ZabbixServer T+3s | **HTTP gap**: 0s | **Result**: PASS

**Commands**:
```bash
# NOTE: --force required (see finding below)
kubectl drain ccs-test-worker1 --ignore-daemonsets --delete-emptydir-data --force
# After test:
kubectl uncordon ccs-test-worker1
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | Drain starts, node cordoned |
| T+1s | CNPG detects primary unreachable; zabbix-pg-2 promoted |
| T+2s | Evictions begin: pg-1, pooler, server, jmx, webservice |
| T+3s | ZabbixServer leader pod evicted; standby (worker2) acquires Lease |
| T+3s | Operator updates EPS to standby IP — traffic restored |

**Lease transitions**: 7 → 8

**Caveats**:
1. `--force` flag required: A missing Tenable security DaemonSet causes standard drain to error.
   Add `--force` to skip missing DaemonSet validation.
2. Parallel failure: CNPG primary and ZabbixServer leader both on worker1; both failed
   simultaneously without any HTTP disruption. Demonstrates system robustness under
   compounded failure.
3. PDB protection worked: `zabbix-pg-primary` PDB (ALLOWED_DISRUPTIONS=0) caused CNPG to
   promote a replica before evicting the primary pod.

---

### Scenario 5 — CNPG Database Primary Failover

**Failure type**: Force-kill CNPG primary pod
**Expected RTO**: 15-30s | **Actual RTO**: ~3s | **HTTP gap**: 0s | **Result**: PASS

**Commands**:
```bash
PRI=$(kubectl get cluster zabbix-pg -n zabbix-suite \
  -o jsonpath='{.status.currentPrimary}')
kubectl delete pod "$PRI" -n zabbix-suite --force --grace-period=0
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | Primary pod (zabbix-pg-2) force-killed |
| T+3s | CNPG promotes replica (zabbix-pg-1) to primary |
| T+3s | zabbix-pg-rw Service switches to new primary |
| T+3s | ZabbixServer reconnects via PgBouncer to new primary |

**Key finding**: CNPG failover is 3s — dramatically faster than expected. Force-deleting the pod
triggers immediate replica promotion without waiting for leaseDuration TTL.

**Post-test issue**: After scenarios 4+5 (2 CNPG primary switches), DatabaseFlapping=True and
DatabaseReady=False. See Known Issues section.

---

### Scenario 6 — PgBouncer Pod Kill

**Failure type**: Force-kill one of two pooler pods
**Expected RTO**: 5-10s | **Actual RTO**: ~2s (pod replaced) | **HTTP gap**: 0s | **Result**: PASS

**Commands**:
```bash
POOLER=$(kubectl get pods -n zabbix-suite \
  -l cnpg.io/poolerName=zabbix-pg-pooler-rw -o name | head -1 | cut -d/ -f2)
kubectl delete pod "$POOLER" -n zabbix-suite --force --grace-period=0
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | Pooler pod killed |
| T+0s | kube-proxy routes to surviving pooler pod immediately |
| T+2s | Deployment creates replacement pod (Running) |
| T+0s-T+30s | HTTP 200 continuous; no Zabbix errors |

**Key finding**: Zero impact. The 2-replica PgBouncer Deployment provides transparent
connection failover. kube-proxy routes existing connections to the survivor.

---

### Scenario 7 — Operator Leader Pod Kill

**Failure type**: Force-kill the active operator pod
**Expected RTO**: ZabbixServer unaffected; operator self-heals in ~10-15s
**Actual RTO**: ~20s (operator restored to 2 pods) | **ZabbixServer**: Unaffected | **Result**: PASS

**Commands**:
```bash
OP_POD=$(kubectl get pods -n zabbix-operator -o name | head -1 | cut -d/ -f2)
kubectl delete pod -n zabbix-operator "$OP_POD" --force --grace-period=0
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | Operator leader pod killed |
| T+0s | Standby operator pod wins leader election (controller-runtime) |
| T+0s | Reconcile loops resume; EPS and Lease unchanged |
| T+20s | Replacement pod Running; operator back to 2 replicas |

**Key finding**: ZabbixServer lease transitions unchanged (=8); EPS unchanged. The operator's
own leader election (Lease `8cf5c4f2.zabbix.io`) is independent of ZabbixServer's Lease.
State is fully recovered from Kubernetes API (no in-memory state lost).

---

### Scenario 8 — Scale ZabbixServer Replicas 0 → 2 (Cold Start)

**Failure type**: Scale to 0 (full stop), then restart
**Expected RTO**: 40-60s | **Actual RTO**: ~24s (pods ready) | **HTTP gap**: 0s | **Result**: PASS

**Commands**:
```bash
kubectl scale deployment my-zabbix-server -n zabbix-suite --replicas=0
kubectl wait --for=delete pod -n zabbix-suite \
  -l app.kubernetes.io/name=zabbix-server --timeout=60s
kubectl scale deployment my-zabbix-server -n zabbix-suite --replicas=2
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | Scale to 0; both pods terminated |
| T+0s-T+60s | EPS NOT cleared (operator reconcile lag; old EPS remains stale) |
| T+24s | New pods reach 2/2 Ready; first pod acquires Lease (trans 8 -> 9) |
| T+~60s | Operator reconcile; EPS updated to new leader IP (10.244.3.174) |
| T+0s-T+60s | HTTP 200 continuous |

**Key finding**: EPS was NOT cleared immediately when pods scaled to 0. The old IP remained
stale in the EPS for up to 60s. HTTP stayed 200 because ZabbixWeb serves from its own DB
connection independently of the ZabbixServer EPS. Zabbix data collection would have been
interrupted during the ~24s pod startup gap.

**Recommendation**: Reduce RequeueAfter from 60s to 10s in zabbixserver_controller.go for
faster stale EPS cleanup after pod termination.

---

### Scenario 9 — ZabbixWeb Pod Kill (Frontend HA)

**Failure type**: Force-kill one of two ZabbixWeb pods
**Expected RTO**: 0-4s | **Actual RTO**: 0s (zero downtime) | **HTTP gap**: 0s | **Result**: PASS

**Commands**:
```bash
WEB=$(kubectl get pods -n zabbix-suite -l app.kubernetes.io/name=zabbix-web \
  -o name | head -1 | cut -d/ -f2)
kubectl delete pod -n zabbix-suite "$WEB" --force --grace-period=0
```

**Actual timeline**:
| T | Event |
|---|-------|
| T+0s | Web pod killed |
| T+0s | Traefik routes to surviving pod — HTTP 200 continuous |
| T+2s | Replacement pod scheduled |
| T+68s | New pod 1/1 Ready |
| T+0s-T+30s | HTTP 200 throughout; no gap at any probe interval |

**Key finding**: Perfect HA. Traefik detects endpoint removal instantly. No user-visible
disruption. Validates the 2-replica web frontend design.

---

## 5. RTO Summary Table

| # | Scenario | Expected RTO | Actual RTO | HTTP Gap | Result |
|---|----------|-------------|------------|----------|--------|
| 1 | Graceful leader pod delete | 5-10s | ~1s | 0s | PASS |
| 2 | Force-kill leader pod (SIGKILL) | 20-25s | ~2s | 0s | PASS |
| 3 | Kill both server pods | 45-60s | ~18s pods ready | 0s | PASS |
| 4 | Worker node drain | 30-50s | ~3s (leader) | 0s | PASS |
| 5 | CNPG primary failover | 15-30s | ~3s | 0s | PASS |
| 6 | PgBouncer pod kill | 5-10s | ~2s | 0s | PASS |
| 7 | Operator leader pod kill | 10-15s | ~20s (self) | 0s | PASS |
| 8 | Scale 0 to 2 cold start | 40-60s | ~24s pods ready | 0s | PASS |
| 9 | ZabbixWeb pod kill | 0-4s | 0s | 0s | PASS |

**All 9 scenarios PASSED. Zero HTTP downtime across all tests.**

---

## 6. Known Issues and Findings

### KI-1: flapCount Stuck After Multiple CNPG Failovers

**Symptom**: `ZabbixDatabase READY=False`, `DatabaseFlapping=True` — "Primary changed N times
(threshold: 2)". The DB gate then blocks EPS updates, causing ZabbixServer to route to no
endpoints.

**Root cause**: `ZabbixDatabase.status.flapCount` increments on each CNPG primary switch but
has no automatic reset mechanism. After `flapCount >= flapThresholdCount` (default: 2), the
controller enters an early-return loop and never re-evaluates SchemaReady/DatabaseReady.

**Workaround** (run after planned maintenance or testing):
```bash
kubectl patch zabbixdatabase my-zabbix-db -n zabbix-suite \
  --subresource=status --type=merge -p '{"status":{"flapCount":0}}'
```

**Recommendation**: Implement a sliding time window for flapCount (reset to 0 after N minutes
of stable primary). Or increase `spec.flapThresholdCount` for maintenance windows.

### KI-2: Standard Drain Blocked by Missing DaemonSet

**Symptom**: `kubectl drain --ignore-daemonsets` errors: "cannot delete daemonsets.apps not
found: tenable-cloud-security-kubernetes-cluster/..."

**Fix**: Always add `--force` to drain commands in this environment:
```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
```

### KI-3: EPS Not Cleared Immediately When Pods Terminate

**Symptom**: Stale EPS IP remains up to 60s after pod termination; traffic to that IP returns
connection refused.

**Impact**: Low for web UI (independent DB connection). Monitoring collection stops during gap.

**Recommendation**: Reduce `RequeueAfter` in `zabbixserver_controller.go` from 60s to 10s.

### Finding: Force-Kill RTO Matches Graceful on Managed Kubernetes

Force-kill RTO (~2s) is nearly identical to graceful (~1s) when pods are deleted from the API.
Kubernetes removes the pod object immediately on force-delete, allowing the standby's lease
election to acquire without waiting for full TTL. Only a true node crash (node unreachable)
requires the full 20s leaseDuration.

---

## 7. Prometheus Alerting Rules

```yaml
groups:
  - name: zabbix-operator
    rules:
      - alert: ZabbixLeaderFlapping
        expr: increase(zabbix_leader_transitions_total[10m]) > 3
        for: 0m
        labels: { severity: warning }
        annotations:
          summary: "ZabbixServer {{ $labels.server }} leader flapping"

      - alert: ZabbixDatabaseNotReady
        expr: zabbix_database_ready == 0
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "ZabbixDatabase {{ $labels.database }} not ready for >2m"

      - alert: ZabbixLeadershipUnstable
        expr: |
          kube_customresource_status_condition{
            resource="zabbixservers",
            condition="LeadershipUnstable",status="true"} == 1
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "ZabbixServer leadership unstable"
```

---

## 8. Operational Procedures

### Force Manual Failover
```bash
LEADER=$(kubectl get lease zabbix-server-leader-my-zabbix-server \
  -n zabbix-suite -o jsonpath='{.spec.holderIdentity}')
kubectl delete pod "$LEADER" -n zabbix-suite
# Verify within 5s:
kubectl get lease zabbix-server-leader-my-zabbix-server -n zabbix-suite \
  -o jsonpath='holder={.spec.holderIdentity} trans={.spec.leaseTransitions}'
```

### Reset Stale flapCount
```bash
kubectl patch zabbixdatabase my-zabbix-db -n zabbix-suite \
  --subresource=status --type=merge -p '{"status":{"flapCount":0}}'
```

### Clear Stale Zabbix HA Node Records
```bash
kubectl exec -n zabbix-suite zabbix-pg-1 -- \
  psql -U postgres -d zabbix -c "DELETE FROM ha_node WHERE status='stopped';"
```

### CNPG Major Version Upgrade (Fully Operator-Controlled)

Patch the CNPG Cluster `spec.imageName` — the operator handles the rest automatically:

```bash
# Example: PG15 → PG16
kubectl patch cluster zabbix-pg -n zabbix-suite --type=merge \
  -p '{"spec":{"imageName":"ghcr.io/cloudnative-pg/postgresql:16"}}'

# Monitor:
kubectl get zabbixdatabase my-zabbix-db -n zabbix-suite -w   # → CNPGUpgrading → Ready
kubectl get zabbixserver my-zabbix-server -n zabbix-suite -w # → CNPGMaintenance → Running
kubectl get pods -n zabbix-suite -w                           # pg_upgrade pod appears
```

**Sequence** (fully automatic, no manual steps):
1. Operator detects CNPG phase changed → `ZabbixDatabase.status.phase = CNPGUpgrading`
2. ZabbixServer detects `DatabaseReady` not fully ready → enters `PhaseCNPGMaintenance`
3. ZabbixServer replicas scaled to 0 (prevents writes during pg_upgrade)
4. CNPG runs pg_upgrade Job on a fresh node (takes 5–15 min for large DBs)
5. CNPG returns to `Ready`; ZabbixDatabase phase returns to `Ready`
6. ZabbixServer auto-restores to original replica count; re-enters `PhaseRunning`

**Tested**: PG15 → PG16 on 2026-03-07. RTO ~12 min (pg_upgrade + pod restart). Zero data loss.

### Credential Secret Rotation

Changing `zabbix-db-creds` content requires manual pod restart — env vars are baked at container
start time (referenced via `secretKeyRef`, not volume mount):

```bash
# 1. Update the secret
kubectl patch secret zabbix-db-creds -n zabbix-suite \
  --type=merge -p '{"stringData":{"password":"NewSecurePassword"}}'

# 2. Rolling restart to pick up new env var
kubectl rollout restart deployment/my-zabbix-server -n zabbix-suite
kubectl rollout restart deployment/my-zabbix-web -n zabbix-suite

# 3. Watch rollout
kubectl rollout status deployment/my-zabbix-server -n zabbix-suite
kubectl rollout status deployment/my-zabbix-web -n zabbix-suite
```

**Note**: TLS CA certificates mounted as volumes (e.g. `zabbix-pg-ca`) are auto-refreshed by
kubelet within ~2 minutes. No manual restart needed for certificate rotation.

### Fresh Install (PhaseSchemaInit)

On a completely fresh database (no schema), the operator automatically detects `RunningVersion == ""`
and enters `PhaseSchemaInit` instead of the upgrade path:

1. ZabbixServer starts in standalone mode (1 replica, `ZBX_STANDALONE_UPGRADE=1`)
2. `schema-init` init container runs: `zcat /usr/share/doc/.../create.sql.gz | psql` (direct PG, not pooler)
3. Schema load verified via `users` table probe; `RunningVersion` set to `Spec.Version`
4. Operator transitions through PostUpgrade → Running

**No manual SQL loading needed for fresh deployments.**

### Check Leader Status
```bash
kubectl describe zabbixserver my-zabbix-server -n zabbix-suite
# Key: Status.CurrentLeader, Status.LeaderSince, Status.ActiveReplicas,
#      Status.RecentLeaderTransitions, Conditions[LeadershipUnstable]
```

---

## 9. Lease Timing Reference

| Scenario | Formula | Measured |
|----------|---------|----------|
| Graceful failover RTO | retryPeriod (~3s) | ~1s |
| Force-kill RTO (pod deleted from API) | ~retryPeriod | ~2s |
| Force-kill RTO (node crash) | leaseDuration | ~20s |
| Supervisor fence time | (leaseDuration - retryPeriod) / 2 | 8.5s |
| Max EPS update lag | retryPeriod + readiness | ~5s after lease |
| Operator reconcile period | RequeueAfter | 60s |

---

## 10. Implementation Status — All Versions

| Item | Implementation | Version |
|------|---------------|---------|
| Prometheus metrics (5 counters/gauges) | `internal/metrics` | v2.7 |
| DB gate for EPS | `DatabaseReady` check in `reconcileEndpointSlice` | v2.7 |
| Liveness probe **REMOVED** | TCP probe on 10051 kills standby — do NOT add back | v2.8 |
| Leadership flapping detection | `RecentLeaderTransitions` + `LeadershipUnstable` condition | v2.7 |
| Status fields | `LeaderSince`, `ActiveReplicas`, `RecentLeaderTransitions` | v2.7 |
| PhaseNodeClean | Clears `ha_node` stale rows before schema upgrade | v2.10 |
| Cascading deletion fix | `zabbix.io/suite-link` finalizer on linked CRs | v2.11 |
| configure-db init container | Zabbix binary reads conf file, not env vars | v2.12 |
| UDP DNS fix (server DB host) | `ZABBIX_PG_POOLER_RW_SERVICE_HOST` ClusterIP env var | v2.13 |
| Web DB host fix | `DB_SERVER_HOST` uses ClusterIP of pooler service | v2.22 |
| Web entrypoint patch | `psql --list` → `pg_isready` (psql 16 / PG 17 mismatch) | v2.23 |
| NodeAddress → service hostname | `ha_node.address = my-zabbix-server-active` | v2.24 |
| PhaseSchemaInit | Fresh-install schema load via `create.sql.gz \| psql` Job | v2.28 |
| Upgrade state machine hardening | Standalone mode, EPS early-return, replica mgmt | v2.14–v2.21 |
| PhaseCNPGMaintenance | Auto-scale to 0 during CNPG major upgrade | v2.25 area |
| RBAC: StatefulSets | `statefulsets.apps` added to ClusterRole (ZabbixProxy cache) | v2.39+ |
| Lease-sidecar `:10051` probe | K8s Lease ≡ Zabbix HA active (no K8s/Zabbix election divergence) | sidecar v2.1 |
| ZBX_SERVER_HOST = DNS name | PHP `isRunning()` string compare: matches `ha_node.address` | v2.41 |
| Cross-CR event watches | DB→Server→Web reactive reconcile chain (seconds vs 60s poll) | v2.41 |
| Secret watches | ZabbixDatabase watches credential/TLS secrets; immediate reconcile | v2.41 |

**Do not add TCP liveness probe on port 10051.** Zabbix standby nodes do NOT listen on 10051.
Process-hang detection is handled exclusively by supervisor heartbeat fencing (SIGKILL if
heartbeat age > 8.5s threshold).

---

*Scenarios 1–9 tested live: 2026-02-28, operator v2.9.*
*CNPG PG15→16 upgrade + Zabbix 7.0.21→7.4 upgrade tested: 2026-03-07, operator v2.41.*
