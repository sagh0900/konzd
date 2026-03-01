# konzd

**Kubernetes Operator Native Distribution**

konzd is a curated, production-ready distribution of Kubernetes Operator CRDs and example
manifests. It provides the declarative API layer — Custom Resource Definitions, annotated
deployment patterns, and reference configurations — without bundling any controller logic.

You bring the operator binary. konzd brings the structure.

---

## Overview

Modern Kubernetes operators expose their functionality through Custom Resource Definitions
(CRDs). Deploying, versioning, and sharing those CRDs consistently across clusters and teams
is non-trivial. konzd solves this by packaging:

- **CRDs** — the API surface of one or more operators, ready to `kubectl apply`
- **Examples** — annotated, copy-paste-ready manifests demonstrating every deployment pattern
- **Reference configs** — field-level documentation embedded directly in YAML comments

konzd follows a convention-over-configuration philosophy: every file is numbered, every
directory is self-describing, and every manifest is annotated with the *why*, not just the
*what*.

### What konzd is NOT

- konzd is **not** an operator runtime — it does not run any controllers
- konzd is **not** a Helm chart — it uses plain Kubernetes YAML for maximum transparency
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
| An operator binary | current | The CRDs alone do nothing without a controller |

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

*konzd — Kubernetes Operator Native Distribution*
*Maintained by SaiKrishna Ghanta, SRE — https://github.com/sagh0900/konzd*
