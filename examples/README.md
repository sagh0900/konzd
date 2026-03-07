# Zabbix Operator — Example Manifests

This directory contains ready-to-use Kubernetes manifests demonstrating the four supported
deployment patterns for the Zabbix Operator. Each subdirectory is self-contained and can
be applied sequentially with `kubectl apply -f`.

---

## Deployment Patterns

| Pattern | Directory | Use Case |
|---------|-----------|----------|
| Standalone / Inline | `standalone/inline/` | Single-node dev/lab; all components in one ZabbixSuite |
| Standalone / Link | `standalone/link/` | Dev/lab; components managed by different teams |
| Production HA / Inline | `production-ha/inline/` | Production; HA ZabbixSuite with DB TLS, PSK, all tuning options |
| Production HA / Decentralized | `production-ha/decentralized/` | Production; each component CR owned independently, then linked |

---

## Pattern Explanations

### Standalone / Inline
All Zabbix components (server, web, JMX, SNMPTrapper) are declared **inside** the
ZabbixSuite CR under `spec.server`, `spec.web`, etc. The operator creates and manages
each component. Best for small teams and test environments.

```
ZabbixSuite (inline)
  ├── server spec embedded
  ├── web spec embedded
  ├── jmx spec embedded
  └── snmpTrapper spec embedded
```

### Standalone / Link
Component CRs (ZabbixServer, ZabbixWeb, …) are **pre-created independently**, then
referenced by name in ZabbixSuite via `existingRef`. The operator updates only
`serverRef`/`databaseRef` on linked CRs; everything else in their specs is untouched.
Useful when different teams own different components.

```
ZabbixDatabase (standalone CR)   ZabbixServer (standalone CR)   ZabbixWeb (standalone CR)
        ↑                                  ↑                            ↑
        └─────────────── ZabbixSuite (link mode, existingRef: true) ───┘
```

### Production HA / Inline
Full high-availability inline ZabbixSuite with:
- 3-node CNPG PostgreSQL cluster with PgBouncer pooler
- DB TLS (verify-full) for server↔PostgreSQL connections
- PSK encryption for server↔agent/proxy communications
- Custom credentials Secret
- Operator sidecar imagePullSecrets via ServiceAccount annotation
- ExtraEnv tuning from ConfigMap (pollers, cache sizes)
- LoadBalancer / NodePort Services for external access
- Dedicated RBAC for the zabbix-suite namespace

### Production HA / Decentralized
Same features as HA/Inline, but each component CR is independently managed.
Ideal for platform teams that pre-provision CNPG clusters separately from
application teams that deploy Zabbix workloads.

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
2. **Zabbix Operator** installed (`kubectl apply -f deploy/00-namespaces.yaml -f deploy/01-crds.yaml -f deploy/02-operator-rbac.yaml -f deploy/03-operator-deployment.yaml`)
3. A **ghcr-pull-secret** in the target namespace (or patch the SA as shown above)

---

## Apply Order

Each example directory has numbered files. Apply in order:

```bash
# Example: standalone inline
kubectl apply -f examples/standalone/inline/00-namespace.yaml
kubectl apply -f examples/standalone/inline/01-secrets.yaml
kubectl apply -f examples/standalone/inline/02-cnpg-cluster.yaml
# Wait for CNPG cluster to be ready
kubectl wait cluster zabbix-pg -n zabbix-dev --for=condition=Ready --timeout=300s
kubectl apply -f examples/standalone/inline/03-zabbixsuite.yaml
```
