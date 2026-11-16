# Jenkins CI on Red Hat OpenShift (GitOps)

Declarative deployment of **Jenkins** on **Red Hat OpenShift 4.x** using **Helm** and **Argo CD**.

## Key Customizations for OpenShift

* **OpenShift Security Context Compatibility:** Overrode upstream UID/GID defaults (`runAsUser: null`, `fsGroup: null`) so the Jenkins controller and sidecars run cleanly under OpenShift's `restricted-v2` Security Context Constraints (SCCs).
* **Init Container & Plugin Management:** Configured `overwritePlugins: false` and `overwritePluginsFromImage: false` to prevent non-interactive `cp: overwrite` crashes on pod restarts while preserving downloaded plugins across restarts.
* **Persistent Storage:** Configured a `3Gi` PersistentVolumeClaim (`jenkins-home`) to retain Jenkins jobs, build history, and plugin state.
* **Configuration as Code (JCasC) & Live Reload:** Enabled the `config-reload` sidecar to watch Kubernetes ConfigMaps and automatically reload JCasC settings without restarting the controller pod.
* **Manifest Rendering & GitOps Sync:** Rendered clean Kubernetes manifests via `helm template --skip-tests` into `./manifests` for automated syncing by **Argo CD**.

## Folder Structure

* `custom-values.yaml` — OpenShift-specific Helm overrides (security context, persistence, plugin settings).
* `manifests/` — Pre-rendered Kubernetes manifests managed by Argo CD.

## Deployment & Access Commands

```bash
# Re-render manifests after updating custom-values.yaml
helm template my-jenkins . --namespace jenkins -f custom-values.yaml --skip-tests --output-dir ./manifests

# Retrieve initial Jenkins admin password
oc get secret my-jenkins -n jenkins -o jsonpath='{.data.jenkins-admin-password}' | base64 -d

# Get Jenkins OpenShift Route URL
oc get route -n jenkins
```
