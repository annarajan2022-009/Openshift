## Repository Structure

| Directory | Component | Description |
| :--- | :--- | :--- |
| [`./argocd`](./argocd) | **Argo CD (GitOps)** | Lightweight Helm values (`argocd-values.yaml`), OpenShift Route config, and namespace `ResourceQuota` (`argocd-quota.yaml`). |
| [`./jenkins`](./jenkins) | **Jenkins CI** | OpenShift-hardened Helm values (`jenkins-values.yaml`), pre-rendered manifests (`manifest/`), `jenkins-quota.yaml`, and Argo CD `Application` (`jenkins-app.yaml`). |
| [`./monitoring`](./monitoring) | **Prometheus, Grafana & Loki** | Argo CD `Application` (`ServerSideApply=true`) and custom Helm overrides for `kube-prometheus-stack`, `loki`, and `alloy`. |
| [`./cluster-config`](./cluster-config) | **RHCOS MachineConfigs** | Declarative OS-level kernel (`sysctl`), NTP (`chrony`), and security banner (`/etc/motd`) manifests managed via MCO. |

## Architecture & Key Implementations

* **OpenShift Cluster Provisioning & Governance:** Configured a Red Hat OpenShift 4.x (CRC / OpenShift Local) environment enforcing namespace-level `ResourceQuota` boundaries (CPU, memory, and storage), Security Context Constraints (SCCs), persistent storage (PVCs), and HTTPS Edge OpenShift Routes.
* **GitOps Continuous Delivery (Argo CD):** Bootstrapped **Argo CD** via Helm (`openshift.enabled=true`) tuned for single-cluster footprint optimization, managing cluster applications declaratively with automated sync, pruning, and self-healing.
* **Helm Chart Customization & Security Hardening:** Customized upstream **Helm** charts (`values.yaml` overrides and `helm template` manifest rendering) to comply with OpenShift's `restricted-v2` SCC (`runAsNonRoot`, dynamic UID/GID assignment), resolved read-only root filesystem issues (`HOME=/var/jenkins_home`), and tuned init-container JVM heap sizing (`-Xmx1024m`) alongside ephemeral build agent pod retention (`OnFailure`).
