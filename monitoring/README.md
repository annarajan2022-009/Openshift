# OpenShift Monitoring & Logging Stack (Argo CD GitOps)

This repository manages the **Prometheus, Grafana, Alertmanager, Loki, and Grafana Alloy** observability stack on **OpenShift Local (CRC)** using an **Argo CD App-of-Apps** GitOps architecture.

It is specifically tuned for a constrained local lab environment (**100GB disk shared with Jenkins**), enforcing strict **2-hour metric/log retention**, **ephemeral storage ceilings (<4.5Gi total)**, and **OpenShift `restricted-v2` Security Context Constraint (SCC)** compliance.

---

## Architecture & Repository Structure

The stack uses Argo CD **multi-source Applications** (pulling upstream Helm charts while reading custom `values.yaml` files from this repository) orchestrated in dependency order via `sync-wave` annotations:

```text
monitoring/
├── root-monitoring-app.yaml        # Entry-point Argo CD "App of Apps" manifest
├── monitoring-quota.yaml           # Namespace ResourceQuota & LimitRange (disk & RAM caps)
├── apps/                           # Child Argo CD Applications (watched by monitoring-root)
│   ├── alloy-rbac-app.yaml         # Wave 1: OpenShift SCC RoleBinding for Alloy
│   ├── loki-app.yaml               # Wave 2: Grafana Loki log aggregation (SingleBinary)
│   ├── alloy-app.yaml              # Wave 3: Grafana Alloy log collector (DaemonSet)
│   └── kube-prometheus-app.yaml    # Wave 3: Prometheus, Grafana, Alertmanager & kube-state-metrics
├── rbac/
│   └── alloy-scc-rbac.yaml         # Grants privileged SCC to Alloy for /var/log/pods access
└── values/
    ├── prom-grafana-values.yaml    # Prometheus, Grafana & Alertmanager config
    ├── loki-values.yaml            # Loki config (2h retention, 10m compaction)
    └── alloy-values.yaml           # Alloy log discovery & Loki push pipeline
```

---

## Key Optimizations Implemented

### 1. Disk Pressure Prevention (100GB CRC Lab)
* **2-Hour Retention Window:** Both Prometheus (`retention: 2h`, `retentionSize: "900MB"`) and Loki (`retention_period: 2h`, `compaction_interval: 10m`) automatically purge data older than 2 hours.
* **Hard `emptyDir` & `ephemeral-storage` Caps:** Every workload has explicit `ephemeral-storage` limits and `emptyDir.sizeLimit` boundaries (`1200Mi` for Prometheus, `1Gi` for Loki, `256Mi` for Alloy, `128Mi` for Alertmanager) to prevent node `DiskPressure` taints.
* **Namespace Guardrails (`ResourceQuota` & `LimitRange`):** Enforces a hard ceiling of `4.5Gi` ephemeral storage across the `monitoring` namespace and automatically injects default CPU/memory/storage limits into any sidecars (such as `prometheus-config-reloader`).

### 2. OpenShift Security Context Constraints (SCC) Compliance
* **Dynamic UID/FSGroup Allocation:** Set `runAsUser: null`, `runAsGroup: null`, and `fsGroup: null` across Prometheus, Grafana, Alertmanager, Loki, and `kube-state-metrics` to satisfy OpenShift's default `restricted-v2` SCC.
* **Disabled Incompatible/Redundant Components:** Disabled `nodeExporter`, `kubeEtcd`, `kubeProxy`, `kubeScheduler`, `kubeControllerManager`, `initChownData`, and admission webhook patch jobs (which conflict with OpenShift's built-in monitoring and SCCs).
* **Declarative RBAC for Alloy:** Bound the `alloy` ServiceAccount to `system:openshift:scc:privileged` via a Git-managed `RoleBinding` (`alloy-scc-rbac.yaml`) so Alloy can read container logs from `/var/log/pods`.

### 3. Integrated Visualization & Routing
* **Pre-wired Loki Datasource:** Grafana automatically provisions `http://loki:3100` as a native datasource via `additionalDataSources`—no manual UI configuration required.
* **OpenShift Edge TLS Route:** Exposes Grafana externally at `https://apps-crc.testing` (default credentials: `admin` / `admin`).

---

## Deployment Steps

### 1. Create Namespace & Apply Resource Guardrails
```bash
oc new-project monitoring
oc apply -f monitoring/monitoring-quota.yaml
```

### 2. Deploy the Argo CD Root Application
```bash
oc apply -f monitoring/root-monitoring-app.yaml
```

### 3. Verify Deployment Health
```bash
# Check Argo CD sync status across all applications
oc get applications -n argocd

# Watch pods start in sync-wave order
oc get pods -n monitoring -w
```

---

## Day-2 Maintenance Commands (CRC Disk Hygiene)
Because Jenkins runs on the same 100GB CRC disk, periodically prune unused container images if disk usage climbs:
```bash
oc adm prune images --confirm
```


---

## Accessing Grafana & Querying Logs

1. **Open Grafana:** Navigate to `https://apps-crc.testing` and log in with `admin` / `admin`.
2. **View Metrics:** Go to **Dashboards** to view pre-built Kubernetes cluster, namespace, and pod resource dashboards powered by **Prometheus**.
3. **Query Container Logs:** Go to **Explore** -> select **Loki** from the top datasource dropdown -> run LogQL queries such as:
   * `{namespace="monitoring"}` — view logs from the monitoring stack
   * `{namespace="jenkins"}` — view logs from your Jenkins instance
   * `{pod=~".*"} |= "error"` — search for errors across all namespaces

---

## Troubleshooting & Common Commands

### Force Hard Refresh in Argo CD
If you push changes to GitHub and want Argo CD to sync immediately without waiting for its 3-minute polling interval:
```bash
oc annotate application monitoring-root loki alloy kube-prometheus-stack -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### Check ResourceQuota & Ephemeral Disk Usage
Verify how much CPU, memory, and ephemeral disk space is currently consumed in `monitoring`:
```bash
oc describe resourcequota monitoring-quota -n monitoring
oc adm top pods -n monitoring
```

### Diagnose Blocked or Pending Pods
If a pod fails to start due to quota or SCC restrictions, check recent namespace events:
```bash
oc get events -n monitoring --sort-by='.lastTimestamp' | tail -n 15
```

### Day-2 CRC Disk Hygiene
Because Jenkins runs on the same 100GB CRC disk, periodically prune unused container images and check node disk pressure if builds slow down:
```bash
oc adm prune images --confirm
oc describe node crc | grep -A 5 "Conditions:"
```