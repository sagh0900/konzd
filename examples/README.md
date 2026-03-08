# Zabbix Operator — Example Manifests

This directory contains ready-to-use Kubernetes manifests demonstrating the five supported
deployment patterns for the Zabbix Operator. Each subdirectory is self-contained and can
be applied sequentially with `kubectl apply -f`.

---

## Critical API Notes

### `spec.cnpg` — not `spec.database`

The ZabbixSuite CNPG backend section uses the key `cnpg:`, **not** `database:`:

```yaml
spec:
  cnpg:                          # ← correct
    clusterRef: {...}
    poolerRef: {...}
    directServiceName: zabbix-pg-rw
    trustWindow: "1h"
```

### ZabbixDatabase has NO link mode

`ZabbixDatabase` is **always** created and managed by the ZabbixSuite from the `cnpg:`
section. There is no `database.existingRef` — the field does not exist. The suite always
creates `<suite-name>-db` and manages it.

### Link mode uses `name:`, not `existingRef.name:`

```yaml
# CORRECT — link mode
server:
  name: my-zabbix-server    # references a pre-existing ZabbixServer CR

# WRONG — does not exist
server:
  existingRef:
    name: my-zabbix-server
```

### Inline mode wraps specs under `.spec`

```yaml
# CORRECT — inline mode
server:
  spec:                     # ← required nesting
    image: "zabbix/..."
    replicas: 2

# WRONG — fields not at component level
server:
  image: "zabbix/..."
  replicas: 2
```

### Proxies and Agents in link mode use `existingRef: true` (flat field)

```yaml
# CORRECT
agents:
  - name: my-zabbix-agent
    existingRef: true       # flat bool on the list entry

# WRONG
agents:
  - existingRef:
      name: my-zabbix-agent
```

---

## Deployment Patterns

| Pattern | Directory | Use Case |
|---------|-----------|----------|
| Standalone / Inline | `standalone/inline/` | Single-node dev/lab; all components in one ZabbixSuite |
| Standalone / Link | `standalone/link/` | Dev/lab; components managed by different teams |
| Production HA / Inline | `production-ha/inline/` | Production; HA ZabbixSuite with DB TLS, PSK, all tuning options |
| Production HA / Decentralized | `production-ha/decentralized/` | Production; each component CR owned independently, then linked |
| Lifecycle Test | `lifecycle-test/` | Verified end-to-end: fresh install → CNPG major upgrade → Zabbix upgrade |

---

## Pattern Explanations

### Standalone / Inline
All Zabbix components (server, web, JMX, SNMPTrapper) are declared **inside** the
ZabbixSuite CR under `spec.server.spec`, `spec.web.spec`, etc. The operator creates and
manages each component. Best for small teams and test environments.

```
ZabbixSuite (inline)
  ├── server.spec embedded
  ├── web.spec embedded
  ├── jmx.spec embedded (optional)
  └── snmpTrapper.spec embedded (optional)
```

### Standalone / Link
Component CRs (ZabbixServer, ZabbixWeb, …) are **pre-created independently**, then
referenced by name in ZabbixSuite via `name:`. The operator updates only
`serverRef`/`databaseRef` on linked CRs; everything else in their specs is untouched.
ZabbixDatabase is always suite-managed from the `cnpg:` section.

```
ZabbixServer (standalone CR)   ZabbixWeb (standalone CR)
         ↑                              ↑
         └──────── ZabbixSuite (link mode, name: "my-zabbix-server") ───────┘
         Suite also creates: ZabbixDatabase (always, from cnpg: section)
```

### Production HA / Inline
Full high-availability inline ZabbixSuite with:
- 3-node CNPG PostgreSQL cluster with PgBouncer pooler
- DB TLS (verify-ca) for server↔PostgreSQL connections
- PSK encryption for server↔agent/proxy communications
- ExtraEnv tuning from ConfigMap (pollers, cache sizes, SMTP)
- LoadBalancer Service for external agent/proxy traffic
- HTTPS ingress with TLS termination at ingress controller

### Production HA / Decentralized
Same features as HA/Inline, but each component CR is independently managed by a
separate team. ZabbixSuite acts as a link-mode aggregator. Ideal for platform teams
that pre-provision CNPG clusters separately from application teams.

### Lifecycle Test
A lean, fully verified deployment set. Demonstrates the actual lifecycle tested on a
7-node live cluster:

1. **Fresh install** (Zabbix 7.0.21 on PG15) — schema auto-loaded via PhaseSchemaInit
2. **CNPG major upgrade** (PG15 → PG16) — operator-managed, no manual steps
3. **Zabbix upgrade** (7.0.21 → 7.4.0) — full upgrade state machine, zero manual SQL

Useful as a minimal reference and integration test harness.

---

## imagePullSecrets — Important Note

The operator sidecar images (`ghcr.io/sagh0900/zabbix-supervisor`,
`ghcr.io/sagh0900/zabbix-lease-sidecar`, `ghcr.io/sagh0900/zabbix-migration-sentinel`)
are pulled from GitHub Container Registry (ghcr.io), which requires authentication.

**The CRDs do NOT expose `imagePullSecrets` as a spec field.** Instead, patch the
`imagePullSecrets` onto the ServiceAccount used by ZabbixServer pods so Kubernetes
handles the credential automatically:

```bash
# Create the pull secret
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USER \
  --docker-password=YOUR_PAT \
  --namespace=zabbix-suite

# Patch the ZabbixServer ServiceAccount (created by the operator on first reconcile)
kubectl patch serviceaccount zabbix-server \
  -n zabbix-suite \
  --type=json \
  -p='[{"op":"add","path":"/imagePullSecrets","value":[{"name":"ghcr-pull-secret"}]}]'
```

Alternatively, use the pre-apply RBAC files in each example directory that include
a ServiceAccount with `imagePullSecrets` pre-configured.

---

## Common Prerequisites

All patterns require:
1. **CloudNativePG operator** installed in the cluster
2. **Zabbix Operator** deployed and running (see main README for deploy instructions)
3. A **ghcr-pull-secret** in the target namespace (or patch the SA as shown above)

---

## Apply Order

Each example directory has numbered files. Apply in order:

```bash
# Example: lifecycle-test (lean, verified, recommended starting point)
kubectl apply -f examples/lifecycle-test/00-namespace.yaml

# Pre-create CNPG cluster + pooler (reference your cluster's manifests)
# kubectl apply -f 04-cnpg-cluster.yaml
# kubectl apply -f 05-cnpg-pooler.yaml
kubectl wait cluster zabbix-pg -n zabbix-suite --for=condition=Ready --timeout=300s

# Apply standalone component CRs first (ZabbixSuite links to them)
kubectl apply -f examples/lifecycle-test/01-zabbixserver.yaml
kubectl apply -f examples/lifecycle-test/02-zabbixweb.yaml
kubectl apply -f examples/lifecycle-test/03-zabbixwebservice.yaml

# Apply ZabbixSuite (creates DB + inline agent, links to above CRs)
kubectl apply -f examples/lifecycle-test/04-zabbixsuite.yaml

# Watch until Ready
kubectl get zabbixsuite my-zabbix -n zabbix-suite -w
```

---

## What's New — v2.46

### Staged HA Startup in PostUpgrade

`reconcilePostUpgrade` now uses a two-step startup strategy:

1. **Start 1 replica** in HA mode. Wait for it to become `Available`.
   - If it crashes (bad image, schema not upgraded), caught immediately with
     minimal blast radius. Only 1 pod fails, not all of them simultaneously.
2. **Scale to `spec.replicas`** once the first pod confirms DB connectivity.

This prevents cascading `CrashLoopBackOff` across all pods after an upgrade completes.
The `replicas:` field in your CR is unchanged — the operator temporarily manages the
count during the PostUpgrade phase.

### Post-Install Web Readiness Gate

The post-install job (`disableDefaultAgentHost: true`) now waits for the ZabbixWeb
Deployment to have `AvailableReplicas >= 1` before being created. This ensures
PHP-FPM is ready to serve the Zabbix JSON-RPC API call. Previously the job could fire
immediately after `upgradePhase=Running` while the web pod was still starting, causing
wasted retries.

Set `status.postInstallDone: false` and delete the Job to re-trigger if needed:
```bash
kubectl patch zabbixserver my-zabbix-server -n zabbix-suite \
  --subresource=status --type=merge -p '{"status":{"postInstallDone":false}}'
kubectl delete job my-zabbix-server-postinstall -n zabbix-suite
```

---

## What's New — v2.43

### Automatic Pod Restart on CNPG CA Rotation
Server pods are now annotated with a SHA256 hash of the DB CA certificate and client
certificate secret contents:
```
zabbix.io/db-tls-secret-hash: <16-char hex>
```
When CNPG rotates its CA and updates the secret, the hash changes → the Deployment spec
changes → a rolling restart fires automatically. **No manual `kubectl rollout restart` needed.**

Applicable when `spec.database.dbTLS.caSecretRef` or `clientCertSecretRef` is configured.

---

### Post-Install: Remove Default Agent Host (`disableDefaultAgentHost`)

Zabbix ships with a built-in **"Zabbix server"** monitoring host that points to a local
Zabbix agent at `127.0.0.1:10050`. In container-native deployments the agent binary is
present in the image but disabled — this host permanently shows **"agent not reachable"**
in the web UI, which is noise.

Set `spec.disableDefaultAgentHost: true` (or `spec.server.spec.disableDefaultAgentHost: true`
in ZabbixSuite inline mode) to automatically delete this host after first startup:

```yaml
# ZabbixServer CR (standalone / link mode)
spec:
  disableDefaultAgentHost: true

  # Optional: only needed after rotating the Admin password from the Zabbix default.
  # On fresh install (Admin/zabbix), omit this field — factory credentials are used.
  # adminCredentialsSecretRef:
  #   name: zabbix-admin-creds   # Secret keys: username, password
```

**Behavior:**
| Scenario | Result |
|----------|--------|
| Host exists (`"Zabbix server"`) | Deleted via API; `status.postInstallDone: true` |
| Host already deleted | Job exits 0 silently; `status.postInstallDone: true` |
| API auth fails (wrong password) | Job retries 3×; `PostInstallFailed` condition set |
| To re-run | Set `status.postInstallDone: false` + `kubectl delete job <server>-postinstall` |

All secrets files in this directory include a pre-configured `zabbix-admin-creds` template.
