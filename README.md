# Upgrading a Kubernetes Cluster with kubeadm (v1.34.0 → v1.35.0)

End-to-end, zero-downtime upgrade of a production Kubernetes cluster using `kubeadm`, with a live application running on it.

## Overview

| | |
|---|---|
| **Cluster** | 1 control plane (`controlplane`) + 1 worker (`node01`) |
| **From → To** | `v1.34.0` → `v1.35.0` |
| **Workload** | 1 Deployment running `nginx:alpine` |
| **OS** | Ubuntu (apt-based install) |

**Challenge:** Upgrade the cluster without impacting users of the application and without provisioning any new VMs.

**Strategy:** Upgrade one node at a time. Before touching a node, drain it so its workloads move to the other node, upgrade it, then uncordon it.

## Key Concepts

| Term | Meaning |
|---|---|
| **Taint** | Marks a node so pods are repelled from it unless they tolerate it. Determines which nodes can host workloads. |
| **Cordon** | Marks a node unschedulable. Existing pods keep running; no new pods are placed on it. |
| **Drain** | Cordons the node **and** evicts its pods so they are rescheduled elsewhere. |
| **Uncordon** | Removes the restriction so the node can accept new pods again. |

## Upgrade Order

1. **Control plane first.** It manages the cluster and coordinates the worker nodes (kubelet, kube-proxy), and `kube-apiserver` must always be at a version **>=** every other component.
2. **Worker nodes second**, one at a time.

## Prerequisites

- `kubectl` access to the cluster and root/sudo on every node
- SSH access from the control plane to the worker nodes
- At least one other schedulable node to absorb pods while a node is drained
- Convenience alias used throughout: `alias k=kubectl`

---

## Step 1: Explore the Environment

**Current version**

```bash
k get nodes
```

Two nodes, both at `v1.34.0`. Note that the `VERSION` column shows the **kubelet** version.

**Which nodes can host workloads?** Check for taints. Pods will need to move between nodes, so this matters.

```bash
k describe node | grep -i taints
```

Neither node has taints, so **both** can host workloads.

**How many applications are running?**

```bash
k get deploy
```

One Deployment.

**Where are the pods running?**

```bash
k get pods -o wide
```

Pods are spread across both `controlplane` and `node01`.

**Check the versions available to upgrade to**

```bash
kubeadm upgrade plan
```

---

## Step 2: Upgrade the Control Plane

### 2.1 Drain the control plane

```bash
k drain controlplane --ignore-daemonsets
```

> `--ignore-daemonsets` is required. Without it the drain fails because of DaemonSet-managed pods (see [Troubleshooting](#troubleshooting)).

Verify:

```bash
k get pods -o wide   # all pods are now on node01
k get nodes          # controlplane shows "Ready,SchedulingDisabled"
```

### 2.2 Point apt at the v1.35 repository

Confirm the OS (look for `DISTRIB_ID=Ubuntu`):

```bash
cat /etc/*release*
```

Reference: [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) (see *Determine which version to upgrade to*).

Following the docs alone was not enough, because the package repository is pinned to a minor version. Edit the repo file and change the minor version to `v1.35`:

```bash
vim /etc/apt/sources.list.d/kubernetes.list
```

```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /
```

Then list the available versions:

```bash
apt update
apt-cache madison kubeadm
```

### 2.3 Upgrade kubeadm

```bash
apt-get install kubeadm=1.35.0-1.1
```

### 2.4 Plan and apply the upgrade

```bash
kubeadm upgrade plan v1.35.0
kubeadm upgrade apply v1.35.0
```

Enter `y` when prompted to confirm. This can take up to ~5 minutes.

### 2.5 Upgrade and restart the kubelet

```bash
apt-get install kubelet=1.35.0-1.1
systemctl daemon-reload
systemctl restart kubelet
```

Verify:

```bash
k get nodes
```

### 2.6 Uncordon the control plane

```bash
k uncordon controlplane
k get nodes   # controlplane should be "Ready" at v1.35.0
```

**Status so far:** `controlplane` is on `v1.35.0`; `node01` is still on `v1.34.0`.

---

## Step 3: Upgrade the Worker Node

### 3.1 Confirm versions

The worker should be one minor version behind the control plane.

```bash
k get nodes
```

### 3.2 Drain the worker

```bash
k drain node01 --ignore-daemonsets
```

Verify:

```bash
k get nodes   # node01 is cordoned (SchedulingDisabled)
k get pods    # all pods have moved to controlplane
```

### 3.3 SSH into the worker

Get the worker's internal IP, then connect:

```bash
k get nodes -o wide
ssh <INTERNAL-IP-OF-node01>
```

### 3.4 Point apt at the v1.35 repository (on node01)

The repo file on `node01` must also be edited. Change the minor version in the URL to `v1.35`:

```bash
vim /etc/apt/sources.list.d/kubernetes.list
apt update
apt-cache madison kubeadm
```

### 3.5 Upgrade kubeadm, the node config, and the kubelet (on node01)

```bash
apt-get install kubeadm=1.35.0-1.1

# Upgrade the local kubelet configuration
kubeadm upgrade node

apt-get install kubelet=1.35.0-1.1
systemctl daemon-reload
systemctl restart kubelet
```

### 3.6 Return to the control plane and uncordon the worker

Exit the SSH session, then:

```bash
k get nodes          # node01 should now show v1.35.0
k uncordon node01
k get nodes          # both nodes "Ready"
```

**Done.** Both nodes are on `v1.35.0` and the application stayed available throughout.

---

## Generic Worker Upgrade Procedure

For clusters with more workers, or to follow the official docs more closely, repeat this for each worker node. Replace `1.35.x-*` with the target patch version. See [Upgrading Linux nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/).

```bash
# 1. Upgrade kubeadm
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.35.x-*' && \
sudo apt-mark hold kubeadm

# 2. Upgrade the local kubelet configuration
sudo kubeadm upgrade node

# 3. Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.35.x-*' kubectl='1.35.x-*' && \
sudo apt-mark hold kubelet kubectl

# 4. Restart the kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 5. From the control plane: bring the node back
kubectl uncordon <node-name>

# 6. Confirm all nodes are on the expected version
kubectl get nodes -o wide
```

---

## Troubleshooting

**`k drain` fails with an error about DaemonSet-managed pods**

Add `--ignore-daemonsets`:

```bash
k drain controlplane --ignore-daemonsets
```

**The upgrade command from the Kubernetes docs didn't work as expected**

The docs show placeholder versions (e.g. `1.32.x-*`). Replace both the minor version and the patch version with your target, and make sure `/etc/apt/sources.list.d/kubernetes.list` points to the matching `v1.35` repository first. Use `apt-cache madison kubeadm` to find the exact package string (here, `1.35.0-1.1`).

---

## Quick Reference

```bash
# Inspect
k get nodes -o wide
k get pods -o wide
k describe node | grep -i taints
kubeadm upgrade plan

# Control plane
k drain controlplane --ignore-daemonsets
kubeadm upgrade apply v1.35.0
k uncordon controlplane

# Worker
k drain node01 --ignore-daemonsets
kubeadm upgrade node
k uncordon node01
```
