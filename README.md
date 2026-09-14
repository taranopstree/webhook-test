# VictoriaMetrics Standalone — Helm Chart

Reference documentation for the **single-node VictoriaMetrics observability stack** currently deployed for this environment. This chart provisions a complete monitoring solution — metrics storage, scraping, alerting, and visualization — managed declaratively through the VictoriaMetrics Operator.

> **Status:** This chart is already deployed and running. The sections below document the current configuration and provide the commands needed to redeploy, upgrade, or replicate this setup in another environment (e.g. staging, DR, or a new cluster).

---

## What's Included

| Component | Role |
|---|---|
| **VictoriaMetrics Operator** | Manages the full lifecycle of all monitoring components via Kubernetes CRDs |
| **VMSingle** | Metrics storage and query engine (ingest, store, and query in a single service) |
| **VMAgent** | Scrapes metrics from Kubernetes workloads and infrastructure, forwards to storage |
| **VMAlert** | Evaluates alerting and recording rules |
| **Alertmanager** | Routes, groups, and manages fired alerts (Slack, email, webhook, etc.) |
| **Grafana** | Dashboards and visualization layer |
| **Kube State Metrics** | Exposes Kubernetes object state as metrics |
| **Node Exporter** | Exposes node-level hardware and OS metrics |

---

## Chart Information

| Field | Value |
|---|---|
| Chart Name | `vm-standalone` |
| Chart Version | `0.0.5` |
| App Version | `0.69.0` |

### Chart Dependencies

| Dependency | Version | Source |
|---|---|---|
| `victoria-metrics-k8s-stack` | `0.75.0` | VictoriaMetrics official Helm repository |
| `prometheus-operator-crds` | `20.0.1` | Prometheus Community Helm repository |

---

## Component Image Versions (Currently Deployed)

All images are pulled from OpsTree's public registry organization on Quay.io: **[quay.io/organization/opstree](https://quay.io/organization/opstree)**

| Component | Image Repository (Quay.io) | Version | Direct Link |
|---|---|---|---|
| VictoriaMetrics Operator | `opstree/victoriametrics-operator` | `v0.73.0` | [View Tag](https://quay.io/repository/opstree/victoriametrics-operator?tab=tags) |
| Config Reloader | `opstree/victoriametrics-operator` | `config-reloader-v0.73.0` | [View Tag](https://quay.io/repository/opstree/victoriametrics-operator?tab=tags) |
| VMSingle | `opstree/victoriamertics-victoriametrics` | `v1.141.0` | [View Tag](https://quay.io/repository/opstree/victoriamertics-victoriametrics?tab=tags) |
| VMAgent | `opstree/victoriametrics-vmagent` | `v1.141.0` | [View Tag](https://quay.io/repository/opstree/victoriametrics-vmagent?tab=tags) |
| VMAlert | `opstree/victoriametrics-vmalert` | `v1.141.0` | [View Tag](https://quay.io/repository/opstree/victoriametrics-vmalert?tab=tags) |
| Alertmanager | `opstree/alertmanager` | `v0.32.0` | [View Tag](https://quay.io/repository/opstree/alertmanager?tab=tags) |
| Grafana | `opstree/grafana` | `12.4.3` | [View Tag](https://quay.io/repository/opstree/grafana?tab=tags) |
| Kube State Metrics | `opstree/kube-state-metrics` | `2.18.0-debian13` | [View Tag](https://quay.io/repository/opstree/kube-state-metrics?tab=tags) |
| Node Exporter | `opstree/node-exporter` | `1.11.1-alpine3.23` | [View Tag](https://quay.io/repository/opstree/node-exporter?tab=tags) |

> This is OpsTree's **public** registry organization on Quay.io. Image versions are pinned intentionally and updated only after internal validation. Anyone can browse the full tag list and verify the exact version in use via the links above.

---

## Prerequisites

To redeploy or replicate this stack in another environment, ensure the following are available on the target cluster:

- Kubernetes v1.28 or later
- Helm 3.x
- `kubectl` configured with cluster access
- A StorageClass supporting `ReadWriteOnce` persistent volumes
- Sufficient node capacity for scheduling (see [Resource Requirements](#resource-requirements) below)

---

## Deployment / Redeployment

### 1. Add Required Helm Repositories

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. Fetch Chart Dependencies

```bash
helm dependency update
```

### 3. Review `values.yaml`

The following values are environment-specific and should be reviewed/adjusted before deploying to a new cluster:

| Setting | Purpose | Current Value |
|---|---|---|
| `vm.grafana.adminPassword` | Grafana admin login password | *(set in values.yaml)* |
| `vm.vmsingle.spec.retentionPeriod` | How long metrics are retained | `7d` |
| `vm.vmsingle.spec.storage.resources.requests.storage` | Metrics storage volume size | `10Gi` |
| `vm.vmsingle.spec.storage.storageClassName` | StorageClass for metrics data | `o11y-gp3-storage` |
| `vm.grafana.persistence.storageClassName` | StorageClass for Grafana data | `o11y-gp3-storage` |
| `*.nodeSelector` / `*.tolerations` | Node placement rules | `team: o11y` |

> If deploying to a cluster that doesn't have `o11y-gp3-storage` or the `team: o11y` node label/taint, update these values or provision matching resources before installing.

### 4. Validate the Chart (Dry Run)

```bash
helm template vm-standalone . -f values.yaml --debug
```

Renders all manifests locally without applying anything — useful for catching configuration errors before deployment.

### 5. Install (New Environment)

```bash
helm install vm-standalone . -f values.yaml -n <namespace> --create-namespace
```

### 6. Upgrade (Existing Deployment)

To apply configuration changes to the currently running deployment:

```bash
helm upgrade vm-standalone . -f values.yaml -n <namespace>
```

To preview changes before applying:

```bash
helm diff upgrade vm-standalone . -f values.yaml -n <namespace>
```

### 7. Verify Deployment Health

```bash
kubectl get pods -n <namespace>
```

All pods should be in a `Running` state:

```
NAME                                          READY   STATUS
<release>-grafana-0                           3/3     Running
<release>-kube-state-metrics-...              1/1     Running
<release>-prometheus-node-exporter-...        1/1     Running
<release>-victoria-metrics-operator-...       1/1     Running
vmagent-<release>-...                         2/2     Running
vmalert-<release>-...                         2/2     Running
vmalertmanager-<release>-0                    2/2     Running
vmsingle-<release>-...                        1/1     Running
```

Confirm all managed resources are operational:

```bash
kubectl get vmsingle,vmagent,vmalert,vmalertmanager -n <namespace>
```

---

## Accessing Services

| Service | Port-Forward Command | URL |
|---|---|---|
| VictoriaMetrics UI | `kubectl port-forward -n <namespace> svc/vmsingle-<release> 8428:8428` | `http://localhost:8428/vmui` |
| VMAgent Targets | `kubectl port-forward -n <namespace> svc/vmagent-<release> 8429:8429` | `http://localhost:8429/targets` |
| Grafana | `kubectl port-forward -n <namespace> svc/<release>-grafana 3000:80` | `http://localhost:3000` |

For persistent access, an Ingress or LoadBalancer Service should be configured per environment.

---

## Resource Requirements

Approximate minimum resource footprint for the full stack:

| Component | CPU Request | Memory Request |
|---|---|---|
| VictoriaMetrics Operator | 100m | 300Mi |
| VMSingle | 1000m | 1Gi |
| VMAgent | 100m | 500Mi |
| VMAlert | 100m | 500Mi |
| Alertmanager | 100m | 500Mi |
| Kube State Metrics | 100m | 256Mi |
| Node Exporter | 50m | 32Mi |

Actual usage scales with cluster size, metric cardinality, and scrape frequency.

---

## Data Retention & Storage

- **Metrics retention:** configurable via `vm.vmsingle.spec.retentionPeriod` (currently `7d`)
- **Metrics storage:** persistent volume, currently `10Gi`
- **Grafana storage:** persistent volume, currently `1Gi` (dashboards, users, settings)

Storage sizing should be revisited periodically based on actual metric volume and cardinality growth.

---

## Support

For issues, upgrades, or configuration assistance, please contact the maintaining team.

| Maintainer | Email |
|---|---|
| Varun Kumar | varun.kumar@opstree.com |

---

## Uninstalling

```bash
helm uninstall vm-standalone -n <namespace>
kubectl delete namespace <namespace>
```

> This does not delete persistent volumes by default, depending on the StorageClass reclaim policy. Verify and clean up PVCs manually if required.
