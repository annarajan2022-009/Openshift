# Cluster Observability Stack (Prometheus, Grafana & Loki) on OpenShift

Declarative deployment of **`kube-prometheus-stack`**, **`grafana-community/loki`**, and **`grafana/alloy`** on **Red Hat OpenShift 4.x** via **Argo CD** and **Helm**.

## Architecture & Key Customizations

* **GitOps Helm Deployment:** Configured an Argo CD `Application` pointing directly to the upstream `prometheus-community/kube-prometheus-stack` Helm chart merged with `custom-values.yaml`, using `ServerSideApply=true` for large Prometheus CRDs.
* **Centralized Log Aggregation (Loki & Alloy):** Deployed **Grafana Loki** (`Monolithic` single-binary mode with TSDB filesystem persistence) alongside **Grafana Alloy** (replacing deprecated `promtail` / `loki-stack`) as a privileged DaemonSet to tail CRI-O container logs from `/var/log/pods` across cluster nodes.
* **OpenShift SCC & UID Range Alignment:** Configured `kube-state-metrics` with explicit OpenShift namespace UID/GID ranges (`1000700000`), `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, and dropped Linux capabilities to comply with `restricted-v2` SCC rules. Granted `privileged` SCC to `loki` and `alloy` service accounts for node log access and runtime profiles.
* **Lifecycle & Revision Management:** Set `revisionHistoryLimit: 2` across `grafana`, `kube-state-metrics`, and `prometheusOperator` deployments to keep ReplicaSet history clean and minimize etcd metadata overhead.
* **External Dashboard Access:** Exposed **Grafana** and **Prometheus** via OpenShift Routes for real-time cluster metrics, node health, and centralized pod log visualization in Grafana Explore.

## Folder Structure

* `application.yaml` - Argo CD `Application` manifest with automated sync, self-heal, and `ServerSideApply=true`.
* `custom-values.yaml` - Helm value overrides for `kube-prometheus-stack` OpenShift security contexts, Grafana credentials, and ReplicaSet retention.
* `loki-values.yaml` - Helm value overrides for `grafana-community/loki` (`Monolithic` mode, TSDB v13 schema, 10Gi PVC persistence).
* `alloy-values.yaml` - Helm value overrides and River pipeline config for `grafana/alloy` Kubernetes pod log discovery and Loki shipping.
* `loki-manifests/` - Rendered Kubernetes manifests for Loki deployment tracking.

## Deployment & Access Commands

```bash
# Grant OpenShift SCC permissions for Loki and Alloy
oc adm policy add-scc-to-user privileged -z loki -n monitoring
oc adm policy add-scc-to-user privileged -z alloy -n monitoring

# Deploy Loki and Grafana Alloy via Helm
helm upgrade --install loki grafana/loki -n monitoring -f loki-values.yaml
helm upgrade --install alloy grafana-official/alloy -n monitoring -f alloy-values.yaml

# Expose Grafana and Prometheus via OpenShift Routes
oc expose svc/prometheus-stack-grafana -n monitoring
oc expose svc/prometheus-stack-kube-prom-prometheus -n monitoring

# Get Dashboard URLs
oc get route -n monitoring

# Retrieve Grafana admin password
oc get secret prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d
```

## Roadmap

* [x] Add **Grafana Loki** & **Grafana Alloy** for centralized container log aggregation inside Grafana.
* [ ] Persist the Loki data source declaratively via `additionalDataSources` in `custom-values.yaml`.
