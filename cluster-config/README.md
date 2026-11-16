# OpenShift Cluster-Level OS Configuration (`MachineConfig`)

Declarative management of **Red Hat Enterprise Linux CoreOS (RHCOS)** node configurations on **Red Hat OpenShift 4.x** using the **Machine Config Operator (MCO)** and **Argo CD**.

## Overview

Because RHCOS is an immutable operating system, manual SSH modifications are avoided to prevent configuration drift. All OS-level kernel parameters, time synchronization settings, and security banners are managed declaratively via Ignition-formatted `MachineConfig` manifests.

## Manifests Included

| File | Target Pool | Purpose |
| :--- | :--- | :--- |
| `99-worker-custom-sysctl.yaml` | `worker` | Tunes Linux kernel parameters (`net.core.somaxconn`, `fs.file-max`, `vm.max_map_count`) for high-throughput workloads. |
| `99-master-chrony-ntp.yaml` | `master` | Configures enterprise NTP (`chrony`) servers on control-plane nodes to protect `etcd` quorum and authentication time sync. |
| `99-worker-chrony-ntp.yaml` | `worker` | Configures matching enterprise NTP (`chrony`) servers across worker nodes. |
| `99-master-motd.yaml` | `master` | Enforces a security and compliance login banner (`/etc/motd`) on control-plane nodes. |
| `99-worker-motd.yaml` | `worker` | Enforces a security and compliance login banner (`/etc/motd`) on worker nodes. |

## Production Rollout & Maintenance Best Practices

* **Pool Separation (`master` vs. `worker`):** Separate `MachineConfig` manifests are maintained per `MachineConfigPool` role label (`machineconfiguration.openshift.io/role`) so worker node rolling reboots can be validated before touching control-plane nodes.
* **Controlled Maintenance Windows:** Before applying multiple `MachineConfig` updates in production, pause the target `MachineConfigPool` so MCO compiles a single rendered config and reboots each node only once:
  ```bash
  # Pause worker pool before applying multiple changes
  oc patch mcp/worker --type merge --patch '{"spec":{"paused":true}}'

  # Unpause during approved maintenance window to trigger rolling reboot
  oc patch mcp/worker --type merge --patch '{"spec":{"paused":false}}'
  ```

## Verification Commands

```bash
# Monitor MachineConfigPool rollout status
oc get mcp

# Verify applied configuration directly on a node without SSH
oc debug node/<node-name> -- chroot /host cat /etc/sysctl.d/99-custom-tuning.conf
oc debug node/<node-name> -- chroot /host chronyc sources -v
```
