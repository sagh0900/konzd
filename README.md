# konzd

**Kubernetes Operator Native Zabbix Distribution**

konzd is a curated, production-ready distribution of CRDs and example manifests for running
[Zabbix](https://www.zabbix.com) on Kubernetes via a Kubernetes Operator. It provides the
declarative API layer — Custom Resource Definitions, annotated deployment patterns, and
reference configurations — without bundling any controller logic.

> [!IMPORTANT]
> **Applying these CRDs registers API types in your cluster, but nothing will reconcile
> them.** A `kubectl apply` of a `ZabbixServer` CR without a running controller produces
> exactly one result: a stored object in etcd. No pods, no Services, no EndpointSlices, no
> Zabbix — nothing. The controller is the hard 95% of the work. See
> [The Operator](#the-operator) below before going further.

---

## The Operator

konzd ships the **API contract** (CRDs + example manifests). The **controller** — the
process that watches those CRDs and drives the cluster toward the declared state — is a
separate binary that must be running for any of this to be functional.

### What the controller does

The controller is responsible for every non-trivial operational concern:

| Responsibility | What the controller handles |
|---|---|
| Pod lifecycle | Creates, updates, and deletes Deployments for ZabbixServer, ZabbixWeb, ZabbixProxy, etc. |
| Database provisioning | Reads `ZabbixDatabase` CR → drives CloudNativePG `Cluster` + PgBouncer `Pooler` creation |
| Schema migrations | Runs a deterministic one-shot Job per version bump; gates on success; halts on failure |
| HA traffic routing | Watches Zabbix's `ha_node` active/standby status; updates the selector-less Service EndpointSlice to point at the currently-active pod |
| TLS and credentials | Mounts DB TLS certificates and resolves credential secrets into container env vars |
| Status aggregation | Rolls up all component conditions into `ZabbixSuite.status` for a single-pane health view |
| Prometheus metrics | Exposes leader transition counters, DB readiness gauges, endpoint update counters |

Without the controller running, a `ZabbixSuite` CR is an inert object. The cluster state
does not change. No Zabbix workloads are created.

### Getting the controller

The reference implementation is available at:

```
ghcr.io/sagh0900/zabbix-operator:latest    # current stable
ghcr.io/sagh0900/zabbix-operator:2.42      # pinned version (recommended for production)
```

The operator requires two auxiliary sidecar images injected into each Zabbix Server pod:

| Image | Version | Purpose |
|---|---|---|
| `ghcr.io/sagh0900/zabbix-lease-sidecar` | `2.1` | Acquires K8s Lease; probes `:10051` to ensure K8s Lease ≡ Zabbix HA active |
| `ghcr.io/sagh0900/zabbix-migration-sentinel` | `2.3` | Runs schema migrations; supports `--ha-cleanup-only`, `--probe-version` |
| `ghcr.io/sagh0900/zabbix-supervisor` | `2.0` | Process supervisor; fences deposed Zabbix leader via heartbeat |

Deploy the operator into its own namespace before applying any CRs from konzd:

```bash
# 1. Install CRDs from konzd
kubectl apply -f https://raw.githubusercontent.com/sagh0900/konzd/main/crds/all-crds-bundled.yaml

# 2. Create the operator namespace and RBAC (includes statefulsets.apps — required for ZabbixProxy)
kubectl create namespace zabbix-operator
# (apply your operator RBAC manifest here — ClusterRole, ClusterRoleBinding, ServiceAccount)

# 3. Deploy the operator
kubectl create deployment zabbix-operator \
  --image=ghcr.io/sagh0900/zabbix-operator:2.42 \
  --namespace=zabbix-operator

# 4. Verify it is running and has acquired the controller leader lease
kubectl logs -n zabbix-operator -l app=zabbix-operator | grep "Starting Controller"
```

> [!NOTE]
> The operator image (`ghcr.io/sagh0900/zabbix-operator`) is currently under active
> development. The CRD API surface (this repository) is at `v1alpha1` and may change
> between minor versions. Pin to a specific version tag in production.

### Controller version compatibility

| Operator image | Lease-sidecar | Sentinel | Key features added |
|---|---|---|---|
| `2.42` | `2.1` | `2.3` | Secret/configmap watches; cross-CR reactive reconcile chain |
| `2.41` | `2.1` | `2.3` | ZBX_SERVER_HOST = DNS (matches ha_node.address for PHP isRunning() check) |
| `2.40` | `2.1` | `2.3` | DB_SERVER_HOST = ClusterIP (bypasses UDP DNS for pooler connection) |
| `2.39` | `2.1` | `2.3` | StatefulSet RBAC fix; full decentralized deploy tested |
| `2.28` | `2.0` | `2.3` | PhaseSchemaInit via schema-load Job (create.sql.gz \| psql) |
| `2.24` | `2.0` | `2.3` | NodeAddress = service hostname (fixes ha_node.address vs ZBX_SERVER_HOST) |
| `2.10` | `2.0` | `2.1` | PhaseNodeClean (ha_node cleanup before schema migration) |
| `2.7`  | `2.0` | `2.0` | Prometheus metrics, DB gate, leadership flapping detection |
| `< 2.5` | `2.0` | `2.0` | Missing credential control and DB TLS; **not recommended** |

> **Sidecar v2.1 is required from operator v2.39+.** The lease-sidecar v2.1 probes
> `127.0.0.1:10051` before acquiring the K8s Lease, ensuring the K8s Lease holder is always
> the Zabbix HA active leader. Without v2.1, the K8s Lease and Zabbix ha_node elections can
> diverge, causing the EndpointSlice to route to a standby pod that doesn't listen on 10051.

---

## About Zabbix

[Zabbix](https://www.zabbix.com) is a battle-tested, open-source enterprise monitoring
platform that has been in production use since 2001. It provides comprehensive, real-time
visibility into infrastructure health — from bare-metal servers and network switches to
virtual machines, cloud workloads, containers, and business applications.

### Why Zabbix?

| Capability | Details |
|-----------|---------|
| **Scale** | Handles millions of metrics per second; proven in environments with 100,000+ hosts |
| **Protocol support** | Zabbix Agent/Agent2, SNMP v1/v2c/v3, IPMI, JMX, HTTP checks, custom scripts |
| **Distributed monitoring** | Proxy architecture spans geographically dispersed sites without VPN tunnels |
| **Alerting** | Built-in escalation policies, maintenance windows, and dependency-based suppression |
| **Dashboards & reports** | Web-based dashboards, scheduled PDF reports via headless Chromium (Web Service) |
| **License** | Open-source. Zabbix ≤ 6.x: GPL v2. **Zabbix 7.0+ (LTS): AGPLv3.** The CRD schemas and operator code in this repository are Apache 2.0, but the Zabbix software itself is AGPLv3 — review your organisation's open-source policy before deploying. |

### Zabbix Component Architecture

A full Zabbix deployment consists of several cooperating components, each represented by a
Custom Resource in konzd:

| CRD | Zabbix Component | Role |
|-----|-----------------|------|
| `ZabbixServer` | `zabbix_server` | Core monitoring engine — collects data, evaluates triggers, fires alerts |
| `ZabbixWeb` | `zabbix-web-nginx-pgsql` | Web frontend — REST API + browser dashboard (PHP/Nginx) |
| `ZabbixWebService` | `zabbix_web_service` | Headless Chromium — generates scheduled PDF reports |
| `ZabbixProxy` | `zabbix_proxy` | Distributed collection proxy — offloads polling from the server |
| `ZabbixAgent` | `zabbix_agent2` | Host-side agent — active/passive metric collection on monitored nodes |
| `ZabbixJMX` | `zabbix_java_gateway` | JMX gateway — bridges Zabbix to Java application MBeans |
| `ZabbixSNMPTrapper` | SNMP trap receiver | Receives inbound SNMP traps from network devices |
| `ZabbixDatabase` | CloudNativePG cluster | PostgreSQL backend managed by the CNPG operator |
| `ZabbixSuite` | (aggregate) | Orchestrates all components as a single declarative unit |

### Running Zabbix on Kubernetes — the challenge

Zabbix was originally designed for VM/bare-metal deployments. Running it on Kubernetes
introduces operational complexity that konzd is designed to address:

- **Server HA and traffic routing**: Zabbix 7+ has native built-in HA — multiple
  `zabbix_server` instances run simultaneously; each writes a heartbeat to the `ha_node`
  PostgreSQL table, and Zabbix's own election logic designates one as `active` while others
  remain in `standby`. The operator maintains a **selector-less Service** (`<name>-active`)
  backed by an **operator-managed EndpointSlice** pointing only to the current active pod's IP.
  A `lease-sidecar` container in each server pod acquires a **per-server Kubernetes Lease**
  (v2.1+: only when the local Zabbix process is listening on port 10051, i.e., is the active
  leader). The operator watches this Lease and updates the EndpointSlice within ~3s of
  failover. The `zabbix-operator` namespace also uses a separate Lease for **operator controller
  HA** (ensuring only one operator pod runs the reconcile loop). These are two distinct leases.
  Typical EndpointSlice update latency after Zabbix active-node transition: ~1–5s.
- **Schema migration safety**: Zabbix database schema upgrades must run exactly once per
  version bump; the operator gates the migration Job with deterministic naming and
  idempotent state-machine checks.
- **HA database**: Zabbix requires PostgreSQL; konzd integrates with
  [CloudNativePG](https://cloudnative-pg.io) for automatic primary failover, connection
  pooling (PgBouncer), and streaming replication — all declaratively configured.
- **Distributed proxies**: Remote-site Zabbix Proxies, PSK-encrypted agent connections,
  and SNMP trap receivers are first-class CRDs, not afterthoughts.

---

## Overview

konzd packages the full Zabbix-on-Kubernetes API surface into a versioned, GitOps-friendly
repository. Deploying, versioning, and sharing Zabbix CRDs consistently across clusters and
teams is non-trivial. konzd solves this by providing:

- **CRDs** — the complete Zabbix operator API surface, ready to `kubectl apply`
- **Examples** — annotated, copy-paste-ready manifests for every deployment pattern (dev
  through production HA)
- **Reference configs** — field-level documentation embedded directly in YAML comments,
  covering every option in every Zabbix component CR

konzd follows a convention-over-configuration philosophy: every file is numbered, every
directory is self-describing, and every manifest is annotated with the *why*, not just the
*what*.

### What konzd is NOT

- konzd is **not** an operator runtime — it ships no controller binary and reconciles
  nothing on its own; see [The Operator](#the-operator) for what you need alongside it
- konzd is **not** a Helm chart — it uses plain Kubernetes YAML for maximum transparency
- konzd is **not** a replacement for [Zabbix itself](https://www.zabbix.com/download) — you
  still need the Zabbix container images
- konzd is **not** opinionated about your GitOps tooling — it works with Flux, ArgoCD, or
  plain `kubectl`

---

## Directory Structure

```
konzd/
├── crds/                            # CustomResourceDefinition schemas
│   ├── all-crds-bundled.yaml        # Single-file install (all CRDs combined)
│   ├── zabbixagents.yaml            # ZabbixAgent CRD
│   ├── zabbixdatabases.yaml         # ZabbixDatabase CRD
│   ├── zabbixjmxs.yaml              # ZabbixJMX CRD
│   ├── zabbixproxies.yaml           # ZabbixProxy CRD
│   ├── zabbixservers.yaml           # ZabbixServer CRD
│   ├── zabbixsnmptrappers.yaml      # ZabbixSNMPTrapper CRD
│   ├── zabbixsuites.yaml            # ZabbixSuite CRD
│   ├── zabbixwebservices.yaml       # ZabbixWebService CRD
│   └── zabbixwebs.yaml              # ZabbixWeb CRD
│
├── examples/
│   ├── standalone/
│   │   ├── inline/                  # All-in-one ZabbixSuite, 1 replica, dev-grade
│   │   └── link/                    # Pre-created CRs, linked by ZabbixSuite
│   │
│   ├── production-ha/
│   │   ├── inline/                  # Full HA, DB TLS, verify-ca, ExtraEnv, LB Service
│   │   └── decentralized/           # Every CR owned by a separate team (link mode)
│   │
│   └── README.md                    # Examples index and pattern comparison
│
├── failover-runbook.md              # Production runbook: 9 tested failover scenarios,
│                                    # RTO table, credential rotation, CNPG major upgrade,
│                                    # fresh install (SchemaInit), known issues
│
└── README.md                        # This file
```

---

## Installation

### Prerequisites

| Requirement | Version | Notes |
|------------|---------|-------|
| Kubernetes | ≥ 1.26 | EndpointSlice GA; CRDs use `apiextensions.k8s.io/v1` |
| kubectl | ≥ 1.26 | For applying manifests |
| CloudNativePG operator | ≥ 1.22 | Required for database examples |
| **Zabbix operator** | **2.42+** | **Required — CRDs do nothing without a running controller. See [The Operator](#the-operator).** |

### 1. Install all CRDs (single command)

```bash
kubectl apply -f https://raw.githubusercontent.com/sagh0900/konzd/main/crds/all-crds-bundled.yaml
```

Or from a local clone:

```bash
git clone https://github.com/sagh0900/konzd.git
kubectl apply -f konzd/crds/all-crds-bundled.yaml
```

### 2. Install CRDs individually

```bash
kubectl apply -f konzd/crds/zabbixservers.yaml
kubectl apply -f konzd/crds/zabbixdatabases.yaml
kubectl apply -f konzd/crds/zabbixwebs.yaml
# ... etc
```

### 3. Verify CRDs are registered

```bash
kubectl get crds | grep zabbix.io
```

Expected output:

```
zabbixagents.zabbix.io           2026-03-01T00:00:00Z
zabbixdatabases.zabbix.io        2026-03-01T00:00:00Z
zabbixjmxs.zabbix.io             2026-03-01T00:00:00Z
zabbixproxies.zabbix.io          2026-03-01T00:00:00Z
zabbixservers.zabbix.io          2026-03-01T00:00:00Z
zabbixsnmptrappers.zabbix.io     2026-03-01T00:00:00Z
zabbixsuites.zabbix.io           2026-03-01T00:00:00Z
zabbixwebservices.zabbix.io      2026-03-01T00:00:00Z
zabbixwebs.zabbix.io             2026-03-01T00:00:00Z
```

---

## Usage — Examples

konzd ships four example patterns, each progressively more feature-complete.
Apply them in the numbered order within each directory.

### Pattern 1: Standalone / Inline (dev/lab)

Single ZabbixSuite CR embedding all component specs. Minimal config, 1 replica, no TLS.

```bash
kubectl apply -f konzd/examples/standalone/inline/00-namespace.yaml
kubectl apply -f konzd/examples/standalone/inline/01-secrets.yaml
kubectl apply -f konzd/examples/standalone/inline/02-rbac.yaml
kubectl apply -f konzd/examples/standalone/inline/03-cnpg-cluster.yaml

# Wait for the database cluster to become ready
kubectl wait cluster zabbix-pg -n zabbix-dev \
  --for=condition=Ready --timeout=300s

kubectl apply -f konzd/examples/standalone/inline/04-zabbixsuite.yaml
```

### Pattern 2: Standalone / Link (team-owned CRs)

Component CRs pre-created independently, then referenced by ZabbixSuite. Useful when
different teams own different components.

```bash
# Apply numbered files in order
for f in konzd/examples/standalone/link/0*.yaml; do
  kubectl apply -f "$f"
done
```

### Pattern 3: Production HA / Inline

Full high-availability with 3-node CNPG PostgreSQL, PgBouncer, DB TLS (verify-full with
mTLS), PSK encryption for monitoring traffic, ExtraEnv from ConfigMap, and HTTPS ingress.

```bash
# 1. Infrastructure
kubectl apply -f konzd/examples/production-ha/inline/00-namespace.yaml
kubectl apply -f konzd/examples/production-ha/inline/01-secrets.yaml    # fill in real values first
kubectl apply -f konzd/examples/production-ha/inline/02-configmaps.yaml
kubectl apply -f konzd/examples/production-ha/inline/03-rbac.yaml
kubectl apply -f konzd/examples/production-ha/inline/04-cnpg-cluster.yaml
kubectl apply -f konzd/examples/production-ha/inline/05-cnpg-pooler.yaml

# 2. Wait for database
kubectl wait cluster zabbix-pg -n zabbix-suite \
  --for=condition=Ready --timeout=300s

# 3. Deploy operator stack
kubectl apply -f konzd/examples/production-ha/inline/06-zabbixsuite.yaml
```

### Pattern 4: Production HA / Decentralized

Each CR is owned and applied by a different team. ZabbixSuite acts as a link-mode
aggregator, reading status from independently-managed CRs.

```bash
# Apply all files in numbered order
for f in konzd/examples/production-ha/decentralized/0*.yaml \
         konzd/examples/production-ha/decentralized/1*.yaml; do
  kubectl apply -f "$f"
done
```

### Reference Manifests

The `examples/reference/` directory contains fully-annotated single-CR manifests
documenting every available field. Use these as copy-paste starting points:

```bash
ls konzd/examples/reference/
# 01-zabbixdatabase.yaml    — all ZabbixDatabase fields documented
# 02-zabbixserver.yaml      — all ZabbixServer fields documented (extraEnv, TLS, HA timing)
# 03-zabbixweb.yaml         — all ZabbixWeb fields documented
# ...
```

---

## Deployment Patterns Comparison

| Pattern | Replicas | TLS | DB | Who manages CRs | Best for |
|---------|----------|-----|----|-----------------|----------|
| `standalone/inline` | 1 | none | single-node CNPG | one team, one CR | dev, lab, PoC |
| `standalone/link` | 1 | none | single-node CNPG | separate per-CR | small multi-team |
| `production-ha/inline` | 2+ | DB verify-ca + PSK | 2-node CNPG + PgBouncer | one team, one CR | production, fast iteration |
| `production-ha/decentralized` | 2+ | DB verify-ca + PSK | 2-node CNPG + PgBouncer | separate per-CR (link mode) | production, platform teams |

## Operational Documentation

### Failover Runbook

`failover-runbook.md` in this repository documents all 9 tested failover scenarios against a
live 7-node Kubernetes cluster, with actual RTOs, timelines, and known issues:

| Scenario | RTO | Result |
|---|---|---|
| Graceful leader pod delete | ~1s | PASS |
| Force-kill leader pod | ~2s | PASS |
| Kill both server pods | ~18s (pods ready) | PASS |
| Worker node drain | ~3s | PASS |
| CNPG primary failover | ~3s | PASS |
| PgBouncer pod kill | ~2s | PASS |
| Operator leader pod kill | ~20s (self-heal) | PASS |
| Scale 0→2 cold start | ~24s (pods ready) | PASS |
| ZabbixWeb pod kill | 0s | PASS |

**All 9 scenarios: zero HTTP downtime.**

Also documented in the runbook:
- CNPG major version upgrade (PG15→16): fully operator-controlled, no manual steps
- Zabbix version upgrade (7.0.21→7.4): automatic via `spec.version` patch
- Fresh install (PhaseSchemaInit): `create.sql.gz | psql` Job, no manual SQL
- Credential secret rotation procedure
- Known issues: flapCount, EPS stale lag, drain `--force` requirement

---

## imagePullSecrets

Operator sidecar images may be hosted in a private registry. CRDs do **not** expose an
`imagePullSecrets` field — the correct approach is to patch the ServiceAccount used by
operator-managed pods:

```bash
# Create the pull secret
kubectl create secret docker-registry registry-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_USER \
  --docker-password=YOUR_TOKEN \
  --namespace=YOUR_NAMESPACE

# Patch the ServiceAccount (pre-create or patch after operator first reconcile)
kubectl patch serviceaccount <operator-sa-name> \
  -n YOUR_NAMESPACE \
  --type=json \
  -p='[{"op":"add","path":"/imagePullSecrets","value":[{"name":"registry-pull-secret"}]}]'
```

The `02-rbac.yaml` in each example pattern demonstrates pre-creating the ServiceAccount
with `imagePullSecrets` already attached, avoiding a pod scheduling race condition.

---

## Upgrading CRDs

CRDs are backwards-compatible within the same `v1alpha1` version. To upgrade:

```bash
# Re-apply — Kubernetes merges CRD changes non-destructively
kubectl apply -f konzd/crds/all-crds-bundled.yaml
```

For version-breaking CRD changes (schema changes to existing fields), follow the
[Kubernetes CRD versioning guide](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/).

---

## GitOps Integration

### Flux v2

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: konzd
  namespace: flux-system
spec:
  interval: 1h
  url: https://github.com/sagh0900/konzd
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: konzd-crds
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: konzd
  path: ./crds
  prune: false   # never auto-delete CRDs — protect existing custom resources
  force: true
```

### ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: konzd-crds
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/sagh0900/konzd
    targetRevision: main
    path: crds
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: false   # never delete CRDs automatically
      selfHeal: true
```

---

## Contributing

1. Fork this repository
2. Add or update CRDs under `crds/` — do not modify the `all-crds-bundled.yaml` manually;
   regenerate it by concatenating individual files:
   ```bash
   yq eval-all '. as $item ireduce ({}; . *+ $item)' crds/*.yaml > crds/all-crds-bundled.yaml
   # or simply:
   cat crds/zabbix*.yaml > crds/all-crds-bundled.yaml
   ```
3. Add example manifests under the appropriate `examples/` subdirectory
4. Open a pull request with a clear description of what changed and why

---

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.

---

*konzd — Kubernetes Operator Native Zabbix Distribution*
*Maintained by SaiKrishna Ghanta, SRE — https://github.com/sagh0900/konzd*
*Operator source: https://github.com/sagh0900/zabbix-operator (v2.42)*
