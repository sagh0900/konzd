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
ghcr.io/sagh0900/zabbix-operator:latest   # current stable
ghcr.io/sagh0900/zabbix-operator:2.9      # pinned version (recommended for production)
```

Deploy the operator into its own namespace before applying any CRs from konzd:

```bash
# 1. Install CRDs from konzd
kubectl apply -f https://raw.githubusercontent.com/sagh0900/konzd/main/crds/all-crds-bundled.yaml

# 2. Create the operator namespace and RBAC
kubectl create namespace zabbix-operator
# (apply your operator RBAC manifest here — ClusterRole, ClusterRoleBinding, ServiceAccount)

# 3. Deploy the operator
kubectl create deployment zabbix-operator \
  --image=ghcr.io/sagh0900/zabbix-operator:2.9 \
  --namespace=zabbix-operator

# 4. Verify it is running and has acquired the controller leader lease
kubectl logs -n zabbix-operator -l app=zabbix-operator | grep "Starting Controller"
```

> [!NOTE]
> The operator image (`ghcr.io/sagh0900/zabbix-operator`) is currently under active
> development. The CRD API surface (this repository) is at `v1alpha1` and may change
> between minor versions. Pin to a specific version tag in production.

### Controller version compatibility

| konzd CRDs | Operator image | Notes |
|---|---|---|
| `main` branch | `2.9` | Current — Prometheus metrics, DB gate, ExtraEnv, credential control, DB TLS |
| `main` branch | `2.8` | Previous — same features; TCP liveness probe removed |
| `main` branch | `< 2.5` | Missing credential control and DB TLS; not recommended |

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
  remain in `standby`. The operator's job at the Kubernetes layer is different: it watches
  which pod is currently the Zabbix-elected active node and routes external traffic to it
  by maintaining a selector-less Service backed by an operator-managed EndpointSlice. The
  Kubernetes Lease in the operator is for **operator controller HA** (ensuring only one
  operator replica runs the reconcile loop at a time) — it is not a replacement for Zabbix's
  own ha_node heartbeat mechanism. Typical EndpointSlice update latency after a Zabbix
  active-node transition: ~5–25 s.
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
│   │   ├── inline/                  # Full HA, DB TLS, PSK, ExtraEnv, LB Service
│   │   └── decentralized/           # Every CR owned by a separate team
│   │
│   └── reference/                   # Fully-annotated single-CR reference manifests
│                                    # (field-level documentation for every option)
│
└── README.md                        # This file
```

---

## Installation

### Prerequisites

| Requirement | Version | Notes |
|------------|---------|-------|
| Kubernetes | ≥ 1.25 | CRDs use `apiextensions.k8s.io/v1` |
| kubectl | ≥ 1.25 | For applying manifests |
| CloudNativePG operator | ≥ 1.22 | Required for database examples |
| **Zabbix operator** | **2.9+** | **Required — CRDs do nothing without a running controller. See [The Operator](#the-operator).** |

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
| `production-ha/inline` | 2+ | DB mTLS + PSK | 3-node CNPG + PgBouncer | one team, one CR | production, fast iteration |
| `production-ha/decentralized` | 2+ | DB mTLS + PSK | 3-node CNPG + PgBouncer | separate per-CR | production, platform teams |

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
