# Upgrading a Kubernetes Cluster with kubeadm (v1.34.0 → v1.35.0)
<img width="2080" height="1000" alt="upgrade-flow" src="https://github.com/user-attachments/assets/a6af7313-3d42-4ccb-9a34-405c6db1739d" />
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
<img width="1032" height="181" alt="Screenshot 2026-10-14 133732" src="https://github.com/user-attachments/assets/5e83c903-2ef9-4fff-8db6-580a6eff8c98" />

**Which nodes can host workloads?** Check for taints. Pods will need to move between nodes, so this matters.

```bash
k describe node | grep -i taints
```
Neither node has taints, so **both** can host workloads.
<img width="972" height="132" alt="Screenshot 2026-10-14 133913" src="https://github.com/user-attachments/assets/5f0604b5-fe4c-45bc-ad1b-b8c760c10a23" />

**How many applications are running?**

```bash
k get deploy
```
One Deployment.
<img width="915" height="141" alt="Screenshot 2026-10-14 133818" src="https://github.com/user-attachments/assets/341246eb-dbdb-44ef-a9e1-ca77df957f08" />


**Where are the pods running?**

```bash
k get pods -o wide
```

Pods are spread across both `controlplane` and `node01`.
<img width="1140" height="203" alt="Screenshot 2026-10-14 134507" src="https://github.com/user-attachments/assets/1b273355-c978-4eb0-a066-1f7459c7a0a9" />


**Check the versions available to upgrade to**

```bash
kubeadm upgrade plan
```
<img width="1187" height="78" alt="Screenshot 2026-10-14 134721" src="https://github.com/user-attachments/assets/a82b7bce-f668-4851-9448-dea4c940944d" />
<img width="1057" height="140" alt="Screenshot 2026-10-14 134758" src="https://github.com/user-attachments/assets/3d07965c-55f1-4173-a003-c0682fd5bb99" />
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
<img width="897" height="185" alt="Screenshot 2026-10-14 140100" src="https://github.com/user-attachments/assets/c17e15a9-3eb1-4bcf-8096-b1d23890c91f" />
<img width="818" height="117" alt="Screenshot 2026-10-14 140148" src="https://github.com/user-attachments/assets/27644002-ddc3-4d6b-b1d5-ebe28c78cb18" />





### 2.2 Point apt at the v1.35 repository

Confirm the OS (look for `DISTRIB_ID=Ubuntu`):

```bash
cat /etc/*release*
```

![Uploading Screenshot 2026-10-14 141407.png…]()


Reference: [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) (see *Determine which version to upgrade to*).

Following the docs alone was not enough, because the package repository is pinned to a minor version. Edit the repo file and change the minor version to `v1.35`:

```bash
vim /etc/apt/sources.list.d/kubernetes.list
```

<img width="933" height="107" alt="Screenshot 2026-10-14 142354" src="https://github.com/user-attachments/assets/da4344c9-2a32-4bdf-9830-094f618808d4" />


```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /
```
<img width="932" height="185" alt="Screenshot 2026-10-14 142253" src="https://github.com/user-attachments/assets/74e6704d-607e-474b-a5c8-cd86c69a2a6f" />



Then list the available versions:

```bash
apt update
apt-cache madison kubeadm
```

<img width="1107" height="656" alt="Screenshot 2026-10-14 142442" src="https://github.com/user-attachments/assets/cd29aa95-ff76-4770-8fe6-f205b0f523f9" />



### 2.3 Upgrade kubeadm

```bash
apt-get install kubeadm=1.35.0-1.1
```
<img width="1106" height="410" alt="Screenshot 2026-10-14 142541" src="https://github.com/user-attachments/assets/7f3294a7-b1ce-4379-b8f2-a9d46bfb1b93" />

### 2.4 Plan and apply the upgrade

```bash
kubeadm upgrade plan v1.35.0
kubeadm upgrade apply v1.35.0
```
<img width="1105" height="510" alt="Screenshot 2026-10-14 142721" src="https://github.com/user-attachments/assets/d6385011-7e90-4a44-bc28-af399bd5fbcd" />
<img width="1103" height="225" alt="Screenshot 2026-10-14 142920" src="https://github.com/user-attachments/assets/2c572767-bdef-4add-9514-178916d05cb5" />




Enter `y` when prompted to confirm. This can take up to ~5 minutes.

### 2.5 Upgrade and restart the kubelet

```bash
apt-get install kubelet=1.35.0-1.1
systemctl daemon-reload
systemctl restart kubelet
```
<img width="1106" height="485" alt="Screenshot 2026-10-14 143002" src="https://github.com/user-attachments/assets/0ef69d14-e261-4e44-a352-63ad83504d0c" />

<img width="661" height="185" alt="Screenshot 2026-10-14 143037" src="https://github.com/user-attachments/assets/959124f8-47a4-4b40-b4c3-3ed1dcc3fef4" />




Verify:

```bash
k get nodes
```


<img width="1318" height="192" alt="Screenshot 2026-10-14 143158" src="https://github.com/user-attachments/assets/fc823174-a415-4eac-8a8b-a675401349a2" />


### 2.6 Uncordon the control plane

```bash
k uncordon controlplane
k get nodes   # controlplane should be "Ready" at v1.35.0
```


<img width="595" height="68" alt="Screenshot 2026-10-14 143234" src="https://github.com/user-attachments/assets/1080a37a-609a-4258-ad6c-69c339f5cdd4" />


**Status so far:** `controlplane` is on `v1.35.0`; `node01` is still on `v1.34.0`.
<img width="647" height="138" alt="Screenshot 2026-10-14 143249" src="https://github.com/user-attachments/assets/2ea6a50d-f609-41fa-9cb4-7d0d91c21cba" />


---

## Step 3: Upgrade the Worker Node

### 3.1 Confirm versions

The worker should be one minor version behind the control plane.

```bash
k get nodes
```
<img width="647" height="138" alt="Screenshot 2026-10-14 143249" src="https://github.com/user-attachments/assets/2ea6a50d-f609-41fa-9cb4-7d0d91c21cba" />

### 3.2 Drain the worker

```bash
k drain node01 --ignore-daemonsets
```



Verify:

```bash
k get nodes   # node01 is cordoned (SchedulingDisabled)
k get pods -o wide    # all pods have moved to controlplane
```

<img width="1342" height="101" alt="Screenshot 2026-10-14 151121" src="https://github.com/user-attachments/assets/7efb8a6b-a29a-4cbc-9ac8-4039c968169b" />
<img width="1350" height="182" alt="Screenshot 2026-10-14 150627" src="https://github.com/user-attachments/assets/8cb1efe9-1643-4fc9-bdbb-dac021b7db30" />


### 3.3 SSH into the worker

Get the worker's internal IP, then connect:

```bash
k get nodes -o wide
ssh <INTERNAL-IP-OF-node01>
```
<img width="1372" height="208" alt="Screenshot 2026-10-14 151157" src="https://github.com/user-attachments/assets/834f6167-727a-465c-a3aa-0005078c21ad" />
<img width="1093" height="588" alt="Screenshot 2026-10-14 151231" src="https://github.com/user-attachments/assets/7f65be01-372a-49ea-a873-24360a7ffcf4" />

### 3.4 Point apt at the v1.35 repository (on node01)

The repo file on `node01` must also be edited. Change the minor version in the URL to `v1.35`:

<img width="1086" height="115" alt="Screenshot 2026-10-14 151419" src="https://github.com/user-attachments/assets/781e9609-704b-4ffa-831c-9bcf168de559" />


```bash
vim /etc/apt/sources.list.d/kubernetes.list
apt update
apt-cache madison kubeadm
```
<img width="1127" height="367" alt="Screenshot 2026-10-14 151527" src="https://github.com/user-attachments/assets/79bbda97-80cf-4a92-ae7e-8d774d5a67d0" />




### 3.5 Upgrade kubeadm, the node config, and the kubelet (on node01)

```bash
apt-get install kubeadm=1.35.0-1.1

# Upgrade the local kubelet configuration
kubeadm upgrade node

apt-get install kubelet=1.35.0-1.1
systemctl daemon-reload
systemctl restart kubelet
```

<img width="952" height="367" alt="Screenshot 2026-10-14 151632" src="https://github.com/user-attachments/assets/6da2a984-6e07-45bb-a285-e59b98a9ef9c" />


### 3.6 Return to the control plane and uncordon the worker

Exit the SSH session, then:
<img width="610" height="117" alt="Screenshot 2026-10-14 152101" src="https://github.com/user-attachments/assets/cb2bf41d-fa9a-406b-b8cd-63be518dc1ff" />


```bash
k get nodes          # node01 should now show v1.35.0
k uncordon node01
k get nodes          # both nodes "Ready"
```

<img width="1066" height="142" alt="Screenshot 2026-10-14 152124" src="https://github.com/user-attachments/assets/72512095-f6a1-4d3c-8dfa-5fdc0971a5e8" />

<img width="970" height="295" alt="Screenshot 2026-10-14 152447" src="https://github.com/user-attachments/assets/19fbc79d-29d7-4ba9-b102-e0453d981453" />
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
