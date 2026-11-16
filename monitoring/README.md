# Cluster Observability Stack (Prometheus & Grafana) on OpenShift

Declarative deployment of **`kube-prometheus-stack`** on **Red Hat OpenShift 4.x** via **Argo CD** and **Helm**.

## Architecture & Key Customizations

* **GitOps Helm Deployment:** Configured an Argo CD `Application` pointing directly to the upstream `prometheus-community/kube-prometheus-stack` Helm chart merged with `custom-values.yaml`, using `ServerSideApply=true` for large Prometheus CRDs.
* **OpenShift SCC & UID Range Alignment:** Configured `kube-state-metrics` with explicit OpenShift namespace UID/GID ranges (`1000700000`), `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, and dropped Linux capabilities to comply with `restricted-v2` SCC rules.
* **Lifecycle & Revision Management:** Set `revisionHistoryLimit: 2` across `grafana`, `kube-state-metrics`, and `prometheusOperator` deployments to keep ReplicaSet history clean and minimize etcd metadata overhead.
* **External Dashboard Access:** Exposed **Grafana** and **Prometheus** via OpenShift Routes for real-time cluster metrics, node health, and pod resource visualization.

## Folder Structure

* `application.yaml` — Argo CD `Application` manifest with automated sync, self-heal, and `ServerSideApply=true`.
* `custom-values.yaml` — Helm value overrides for OpenShift security contexts, Grafana credentials, and ReplicaSet retention.

## Deployment & Access Commands

```bash
# Expose Grafana and Prometheus via OpenShift Routes
oc expose svc/prometheus-stack-grafana -n monitoring
oc expose svc/prometheus-stack-kube-prom-prometheus -n monitoring

# Get Dashboard URLs
oc get route -n monitoring

# Retrieve Grafana admin password
oc get secret prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d
```

## Roadmap
* [ ] Add **Grafana Loki** (`loki-stack`) for centralized container log aggregation inside Grafana.
