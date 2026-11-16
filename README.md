# Red Hat OpenShift GitOps & Observability Reference Architecture

End-to-end declarative infrastructure, CI/CD, and observability setup on **Red Hat OpenShift**, managed through **GitOps (Argo CD)** and **Helm**.

## Repository Structure

| Directory | Component | Description |
| :--- | :--- | :--- |
| [`./jenkins`](./jenkins) | **Jenkins CI** | OpenShift-hardened Jenkins Helm values (`custom-values.yaml`), JCasC auto-reload, persistent storage, and rendered Kubernetes manifests. |
| [`./monitoring`](./monitoring) | **Prometheus, Grafana & Loki** | Argo CD `Application` (`ServerSideApply=true`) and custom Helm overrides for `kube-prometheus-stack`, `loki`, and `alloy`. |
| [`./cluster-config`](./cluster-config) | **RHCOS MachineConfigs** | Declarative OS-level kernel (`sysctl`), NTP (`chrony`), and security banner (`/etc/motd`) manifests managed via MCO. |

## Architecture & Key Implementations

* **OpenShift Cluster Provisioning & Configuration:** Installed and configured a Red Hat OpenShift 4.x environment, managing RBAC, Security Context Constraints (SCCs), persistent storage (PVCs), and external application routing via OpenShift Routes.
* **GitOps Continuous Delivery (Argo CD):** Installed and configured **Red Hat OpenShift GitOps (Argo CD)** with automated sync policies, self-healing, and server-side apply to manage cluster workloads declaratively from Git.
* **Helm Chart Customization & Security Hardening:** Customized upstream **Helm** charts (`values.yaml` overrides and template rendering) to comply with OpenShift's strict `restricted-v2` Security Context Constraints (`runAsNonRoot`, UID/GID range mapping, and `readOnlyRootFilesystem`), and configured `privileged` SCC bindings where required for node-level log collection.
* **CI/CD Pipeline Integration (Jenkins & Azure DevOps):**
  * Deployed **Jenkins** on OpenShift with persistent storage and JCasC (Jenkins Configuration as Code) via Argo CD.
  * Integrated **Jenkins** and **Azure DevOps Pipelines** with OpenShift GitOps to decouple container image builds (CI) from declarative cluster deployments (CD).
* **Full-Stack Observability (Prometheus, Grafana & Loki):** Deployed `kube-prometheus-stack` via Argo CD alongside **Grafana Loki** and **Grafana Alloy** for unified cluster metrics, centralized container log aggregation, and external **Grafana** dashboards.

## Roadmap (In Progress)

* [x] Centralized container log aggregation using **Grafana Loki** & **Grafana Alloy**
* [ ] Cluster backup and disaster recovery automation using **Velero / OADP**
* [ ] Encrypted GitOps secrets management with **Sealed Secrets**
