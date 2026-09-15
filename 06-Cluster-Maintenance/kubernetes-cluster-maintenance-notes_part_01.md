# Cluster Maintenance — Complete Notes (Part 1 of 2)

> **Part 1 of 2** — covers Sections 01–06 (Section Introduction, OS Upgrades + Practice
> Test, Kubernetes Software Versions, Cluster Upgrade Introduction + Practice Test).
> Next: [kubernetes-cluster-maintenance-notes_part_02.md](kubernetes-cluster-maintenance-notes_part_02.md)
> (Sections 07–11 + Quick Revision Checklist).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/06-Cluster-Maintenance/`

---

## 01. Cluster Maintenance — Section Introduction

- Video reference: *Cluster Maintenance Section Introduction* (KodeKloud) — https://kodekloud.com/topic/cluster-maintenance-section-introduction-2/
- This section of the CKA course covers three major cluster-maintenance themes:
  - **Cluster Upgrade Process** — how to safely move a running cluster from one Kubernetes minor/patch version to another.
  - **Operating System Upgrades** — how to patch/reboot the underlying node OS without losing application availability.
  - **Backup and Restore Methodologies** — how to protect against data loss (resource configs and the etcd cluster) and how to recover from a disaster.
- Big picture: everything in this section is about **keeping a live cluster healthy and safe** — as opposed to the previous "Application Lifecycle Management" section, which was about managing workloads *on* a healthy cluster. These are core Cluster Administrator (not just Developer) responsibilities, which is why this section is heavily weighted on the CKA exam.

---

## 02. OS Upgrades

- Video reference: https://kodekloud.com/topic/os-upgrades/
- Topic: what happens to a node's workloads when that node needs OS-level maintenance (patching, kernel upgrade, reboot), and how to do that maintenance safely.

### Node failure / node down behavior

- **If a node is down (unreachable/NotReady) for more than 5 minutes, then the Pods on that node are considered lost and are terminated from that node.**
  - This "5 minutes" figure is the Kubernetes controller-manager's default **`pod-eviction-timeout`** — the amount of time the control plane waits after a node is marked `NotReady`/unreachable before it considers the Pods on it for eviction/rescheduling (general Kubernetes knowledge underlying this exact statement).
  - **Inferred context (image `os.PNG`):** the diagram most likely shows a timeline: `Node goes down` → `Node marked NotReady` → `(wait ~5 minutes)` → `Pods on that node marked for deletion / rescheduled onto healthy nodes (if managed by a ReplicaSet/Deployment)`. It probably also implies that if the node comes back online **within** that window, the Pods are left alone and simply continue running — the eviction only fires once the timeout elapses.

### Draining a node (planned maintenance)

- Rather than waiting for an unplanned outage to trigger the 5-minute eviction, you can **purposefully drain** a node before doing maintenance on it, so that its workloads are proactively (and gracefully) moved to other nodes:
  ```
  $ kubectl drain node-1
  ```
  - `drain` **safely evicts** all Pods off the node (respecting PodDisruptionBudgets where configured), and the ReplicaSet/Deployment controllers backing those Pods **recreate** them on other, schedulable nodes.
  - As a side effect of `drain`, the node is **also cordoned** — i.e. automatically marked **unschedulable** — so nothing new gets scheduled onto it while it's being drained/maintained.
- **After the node comes back online following maintenance, it remains marked unschedulable** (the cordon persists) — you must explicitly **uncordon** it to make it schedulable again:
  ```
  $ kubectl uncordon node-1
  ```

### `cordon` vs `drain`

- There is also a separate, standalone command called **`cordon`**.
  - `cordon` simply **marks a node unschedulable** — nothing more.
  - **Unlike `drain`, `cordon` does NOT terminate or move the Pods already running on that node** — existing Pods keep running right where they are; only *future* scheduling onto that node is blocked.
- **Inferred context (image `drain.PNG`):** likely a side-by-side comparison diagram of the two commands:
  - **`cordon`**: node marked `SchedulingDisabled`, existing Pods (`[pod-a][pod-b]`) stay in place, untouched.
  - **`drain`**: node marked `SchedulingDisabled` **and** existing Pods are evicted and rescheduled onto other nodes (`[pod-a]`/`[pod-b]` move to `node-2`/`node-3`), leaving the drained node empty of application workloads.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/

---

## 03. Practice Test — OS Upgrades

Solutions to practice test — OS Upgrades. Step-by-step lab-guide walkthrough:

1. **"Let us explore the environment first. How many nodes do you see in the cluster?"**
   ```bash
   kubectl get nodes
   ```
   - Count the rows returned (each row = one node, master + workers).

2. **"How many applications do you see hosted on the cluster?"**
   ```bash
   kubectl get deploy
   ```
   - Count the Deployments listed — each represents one hosted application.

3. **"Run the command `kubectl get pods -o wide` and get the list of nodes the pods are placed on."**
   ```bash
   kubectl get pods -o wide
   ```
   - The `-o wide` output adds a `NODE` column showing which node each Pod is currently scheduled on — establishes the "before" baseline.

4. **"Run the command `kubectl drain node01 --ignore-daemonsets`."**
   ```bash
   kubectl drain node01 --ignore-daemonsets
   ```
   - `--ignore-daemonsets` is required because DaemonSet-managed Pods are **not evictable** in the normal sense (a DaemonSet Pod is meant to exist on every eligible node, so `drain` cannot legitimately "move" it elsewhere) — without this flag, `drain` refuses to proceed if any DaemonSet Pods are present on the node.
   - This both cordons `node01` and evicts/reschedules its non-DaemonSet Pods.

5. **"Run the command `kubectl get pods -o wide` and get the list of nodes the pods are placed on."**
   ```bash
   kubectl get pods -o wide
   ```
   - Confirms the Pods that were on `node01` have moved to other nodes (the "after" state, contrasted with step 3).

6. **"Run the command `kubectl uncordon node01`."**
   ```bash
   kubectl uncordon node01
   ```
   - Marks `node01` schedulable again.

7. **"Run the command `kubectl get pods -o wide`."**
   ```bash
   kubectl get pods -o wide
   ```
   - Inspect placement again.

8. **"Why are there no pods on node01?"**
   - **Answer:** *Only when new pods are created will they be scheduled [onto node01].*
   - Uncordoning does **not** retroactively rebalance existing Pods back onto the now-schedulable node — it only makes the node eligible for **future** scheduling decisions. The Pods that were moved off during the drain stay on their new nodes until something (a new rollout, a scale-up, a Pod deletion, etc.) causes new Pods to be scheduled.

9. **"Use the command `kubectl describe node master` and look under taint section to check if it has any taints."**
   ```bash
   kubectl describe node master
   ```
   - Look at the `Taints:` field. On a typical single/stacked control-plane setup the master carries the `node-role.kubernetes.io/master:NoSchedule` (or `node-role.kubernetes.io/control-plane:NoSchedule`) taint by default, which is *why* ordinary application Pods don't land on the master node even though it's technically part of the cluster.

10. **"Run the command `kubectl drain node02 --ignore-daemonsets`."**
    ```bash
    kubectl drain node02 --ignore-daemonsets
    ```

11. **"Check the applications hosted on the node02."**
    - **Answer:** *node02 has a pod not part of a replicaset.*
    ```bash
    kubectl get pods -o wide
    ```
    - The drain in step 10 will have **failed/warned** for this Pod, because `kubectl drain` refuses to evict a bare Pod that isn't backed by a ReplicaSet/Deployment/Job/etc. (there is no controller that will recreate it elsewhere) unless you explicitly force it.

12. **"Check the list of pods."**
    ```bash
    kubectl get pods -o wide
    ```

13. **"What would happen to hr-app if node02 is drained forcefully?"**
    ```bash
    kubectl drain node02 --ignore-daemonsets --force
    ```
    - **Answer:** *`hr-app` will be lost forever.* Since `hr-app` is a standalone Pod (not managed by any controller), forcing the drain with `--force` deletes it outright and **nothing recreates it** — there is no ReplicaSet to reschedule it on another node. This is the core exam lesson about `--force`: it lets you evict unmanaged Pods, but at the cost of permanently losing them unless you've backed up their definition separately.

14. **"Run the command `kubectl drain node02 --ignore-daemonsets --force`."**
    ```bash
    kubectl drain node02 --ignore-daemonsets --force
    ```
    - Executes the forced drain, confirming the loss of `hr-app` predicted above.

15. **"Run the command `kubectl cordon node03`."**
    ```bash
    kubectl cordon node03
    ```
    - Marks `node03` unschedulable **without** touching any Pods already running there (contrast with the drains performed on `node01`/`node02`).

**Exam-relevant takeaway:** `kubectl drain` requires `--ignore-daemonsets` whenever DaemonSet Pods exist on the node, and requires `--force` to evict Pods not backed by a controller — but forcing eviction of an unmanaged Pod **permanently destroys it**. `cordon` alone never touches existing Pods. Uncordoning a node does not rebalance already-placed workloads back onto it.

---

## 04. Kubernetes Software Versions

- Video reference: https://kodekloud.com/topic/kubernetes-software-versions/
- Topic: how to read a Kubernetes version number and how the various Kubernetes components/binaries are released.

### Seeing the installed version

```
$ kubectl get nodes
```
- **Inferred context (image `kgn.PNG`):** likely shows sample `kubectl get nodes` output where the `VERSION` column (e.g. `v1.18.0`) reports the **kubelet** version running on each node — this is the same nuance called out later in file 05 (the version shown here is the kubelet's version, not necessarily the API server's).

### Anatomy of a version number

- A Kubernetes version number consists of **3 parts**:
  1. **Major version**
  2. **Minor version**
  3. **Patch version**
  - E.g. for `v1.11.3`: major = `1`, minor = `11`, patch = `3`.
- **Inferred context (image `mmp.PNG`):** likely a labeled breakdown of a sample version string (something like `v1.11.3`) with arrows pointing to each of the three segments and their names (Major / Minor / Patch), exactly matching the bullet list above.

### Kubernetes release procedure

- Kubernetes follows a **standard software release versioning procedure**.
- All Kubernetes releases can be found at https://github.com/kubernetes/kubernetes/releases
- **Inferred context (image `r1.PNG`):** likely a screenshot of the GitHub releases page itself, showing a chronological list of release tags (`v1.19.0`, `v1.18.x`, `v1.17.x`, …), illustrating the pace/cadence of minor releases (historically roughly every ~3 months).
- **Inferred context (image `r2.PNG`):** likely zooms into one release's details/changelog, or illustrates the **pre-release maturity stages** each new minor version's features pass through before a stable release — i.e. **Alpha** (disabled by default, may be buggy/incomplete, feature can be dropped), **Beta** (enabled by default but still evolving, more well-tested), and **Stable/GA** (General Availability — will appear in released software for many subsequent versions) — this is standard Kubernetes feature-maturity terminology commonly paired with this exact release-history discussion.

### What's inside a downloaded release package

- The downloaded Kubernetes release package contains **all the Kubernetes components** in it — **except**:
  - **`ETCD Cluster`**
  - **`CoreDNS`**
  - These two are excluded because they are **separate projects** in their own right (etcd and CoreDNS each have their own independent release cycles/repositories), not part of the core `kubernetes/kubernetes` release artifact.
- **Inferred context (image `r3.PNG`):** likely a diagram of the release tarball's contents — listing the core binaries it *does* include (e.g. `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubelet`, `kube-proxy`, `kubectl`, `kubeadm`) versus `etcd` and `coredns` sitting **outside** the box with a note that they must be sourced/versioned independently.

### References

- https://blog.risingstack.com/the-history-of-kubernetes/
- https://kubernetes.io/docs/setup/release/version-skew-policy/

---

## 05. Cluster Upgrade Introduction

- Video reference: https://kodekloud.com/topic/cluster-upgrade-introduction/

### Do all Kubernetes components need to be on the same version?

- **No — components can be at different release versions.**
- **At any given time, Kubernetes officially supports only the most recent 3 minor versions** (this is the version-skew policy referenced in file 04).
- **The recommended approach is to upgrade one minor version at a time** (e.g. `1.10 → 1.11 → 1.12`, not straight from `1.10 → 1.12`).
- **Inferred context (image `up2.PNG`):** likely a timeline/ladder diagram showing supported version windows — e.g. if the latest release is `v1.13`, the cluster is still "supported" at `v1.13`, `v1.12`, and `v1.11`, but **not** `v1.10` or earlier — and an arrow showing the safe upgrade path stepping through each minor version in sequence rather than skipping any.

### Options to upgrade a Kubernetes cluster

- **Inferred context (image `opt.PNG`):** likely lists the practical ways an administrator can perform a cluster upgrade, most plausibly:
  - Using **`kubeadm upgrade`** (for kubeadm-provisioned clusters) — the path detailed below.
  - Using a **managed Kubernetes service's** upgrade feature (e.g. cloud-provider-managed clusters like GKE/EKS/AKS, where the provider handles much of the control-plane upgrade).
  - **Manually** upgrading each component/binary yourself (for clusters built "the hard way").

### Upgrading a cluster — 2 major steps

- Upgrading a cluster involves **2 major steps**:
  1. **Upgrade the master/control-plane node(s).**
  2. **Upgrade the worker nodes.**

### Worker node upgrade strategies

- There are different strategies available for upgrading the worker nodes:
  1. **Upgrade all at once.** But then all Pods on all nodes are down simultaneously, and users cannot access the applications during the upgrade — this causes downtime.
     - **Inferred context (image `stg1.PNG`):** likely shows all worker nodes going down for upgrade at the same time, with a clear "application unavailable" gap across the whole cluster.
  2. **Upgrade one node at a time.** Move that node's workloads off (drain) before upgrading it, then move on to the next node — the rest of the cluster (and thus the application) stays available throughout.
     - **Inferred context (image `stg2.PNG`):** likely a sequential diagram: node A drained/upgraded/uncordoned while nodes B & C continue serving traffic, then B drained/upgraded/uncordoned while A & C serve traffic, and so on — no global outage, at the cost of the upgrade taking longer overall.
  3. **Add new nodes to the cluster** (already running the new version), migrate workloads onto them, then decommission the old nodes — useful in cloud/elastic-infrastructure environments where you're not limited to a fixed set of VMs.
     - **Inferred context (image `stg3.PNG`):** likely shows new, already-upgraded nodes being added alongside the old cluster, workloads gradually shifting over (similar to a rolling infrastructure replacement / blue-green node pool), and the old nodes being removed once empty.

### `kubeadm` — Upgrade the master node

- `kubeadm` has a built-in **`upgrade`** command that helps upgrade clusters.
  ```
  $ kubeadm upgrade plan
  ```
  - **Inferred context (image `kube1.png`):** likely shows sample `kubeadm upgrade plan` output — it typically reports the current cluster version, the latest stable version available, a compatibility table of component versions (API Server, Controller Manager, Scheduler, kube-proxy, CoreDNS, etcd) and their target versions, plus the exact `kubeadm upgrade apply vX.Y.Z` command recommended to run next.
- Upgrade the `kubeadm` tool itself first, e.g. from v1.11 to v1.12:
  ```
  $ apt-get upgrade -y kubeadm=1.12.0-00
  ```
- Then upgrade the cluster's control-plane components using `kubeadm`:
  ```
  $ kubeadm upgrade apply v1.12.0
  ```
- **After this, if you run `kubectl get nodes`, you will still see the older version reported.** This is because that command's `VERSION` column shows the version of the **kubelet** on each node as registered with the API Server — **not** the version of the API Server itself, which has already been upgraded by `kubeadm upgrade apply`.
  ```
  $ kubectl get nodes
  ```
  - **Inferred context (image `kubeu.PNG`):** likely shows exactly this — `kubectl get nodes` still reporting the old kubelet version string on the master row, right after a successful `kubeadm upgrade apply`, to visually make the point above.
- **Upgrade `kubelet` on the master node itself:**
  ```
  $ apt-get upgrade kubelet=1.12.0-00
  ```
- **Restart the kubelet:**
  ```
  $ systemctl restart kubelet
  ```
- **Run `kubectl get nodes` to verify:**
  ```
  $ kubectl get nodes
  ```
  - **Inferred context (image `kubeu1.PNG`):** likely shows `kubectl get nodes` now reporting the **new** version for the master node, confirming the kubelet restart picked up the upgrade — completing the master-node upgrade sequence: upgrade `kubeadm` → `kubeadm upgrade apply` → upgrade `kubelet` → restart `kubelet` → verify.

### `kubeadm` — Upgrade worker nodes

- From the master node, run `kubectl drain` to move that worker's workloads to other nodes first:
  ```
  $ kubectl drain node-1
  ```
- On the worker node, upgrade the `kubeadm` and `kubelet` packages:
  ```
  $ apt-get upgrade -y kubeadm=1.12.0-00
  $ apt-get upgrade -y kubelet=1.12.0-00
  ```
- Update the node's local configuration for the new kubelet version:
  ```
  $ kubeadm upgrade node config --kubelet-version v1.12.0
  ```
- Restart the kubelet service:
  ```
  $ systemctl restart kubelet
  ```
- Mark the node schedulable again:
  ```
  $ kubectl uncordon node-1
  ```
  - **Inferred context (image `kubeu2.PNG`):** likely a flow diagram of exactly this worker-node sequence — `drain` → upgrade packages → `kubeadm upgrade node config` → restart kubelet → `uncordon` — as a repeatable per-node loop.
- **Upgrade all worker nodes in the same way**, one at a time.
  - **Inferred context (image `kubeu3.PNG`):** likely shows this same per-node loop being repeated across `node-1`, `node-2`, `node-3`, … reinforcing "one node at a time" as the safe, no-downtime worker-upgrade strategy chosen earlier.

### Demo Video

- https://kodekloud.com/topic/demo-cluster-upgrade/

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
- https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/

---

## 06. Practice Test — Cluster Upgrade Process

Solutions to practice test — cluster upgrade process. Step-by-step lab-guide walkthrough:

1. **"What is the current version of the cluster?"**
   ```bash
   kubectl get nodes
   ```
   - Read the `VERSION` column (reporting kubelet versions, effectively the cluster's current version baseline).

2. **"How many nodes are part of this cluster?"**
   ```bash
   kubctl get nodes
   ```
   - *(Note: the source text has a typo, `kubctl` instead of `kubectl` — the intended/correct command is `kubectl get nodes`.)* Count the rows returned.

3. **"Check what nodes the pods are hosted on."**
   ```bash
   kubectl get pods -o wide
   ```
   - Inspect the `NODE` column for each Pod's placement.

4. **"Count the number of deployments."**
   ```bash
   kubectl get deploy
   ```

5. **"Run the command `kubectl get pods -o wide`."**
   ```bash
   kubectl get pods -o wide
   ```
   - A repeat inspection, establishing/reconfirming the pre-upgrade placement baseline.

6. **"You are tasked to upgrade the cluster. User's accessing the applications must not be impacted. And you cannot provision new VMs. What strategy would you use to upgrade the cluster?"**
   - **Answer:** *Upgrade one node at a time while moving the workloads to the other [nodes].*
   - Reasoning: "no new VMs" rules out the "add new nodes" strategy from file 05; "users must not be impacted" rules out "upgrade all at once" (which causes a full outage) — leaving the **rolling, one-node-at-a-time** approach (drain → upgrade → uncordon, repeated per node) as the only viable strategy.

7. **"Run the `kubeadm upgrade plan` command."**
   ```bash
   kubeadm upgrade plan
   ```
   - Reports the current versions of the various control-plane components and what they can safely be upgraded to.

8. **"Run the `kubectl drain master --ignore-daemonsets`."**
   ```bash
   kubectl drain master --ignore-daemonsets
   ```
   - Cordon + evict the control-plane node's evictable Pods before touching its packages, mirroring the worker-node procedure applied here to the master.

9. **"Run the command `apt install kubeadm=1.18.0-00` and then `kubeadm upgrade apply v1.18.0` and then `apt install kubelet=1.18.0-00` to upgrade the kubelet on the master node."**
   ```bash
   apt install kubeadm=1.18.0-00
   kubeadm upgrade apply v1.18.0
   apt install kubelet=1.18.0-00
   ```
   - Same 3-command master-upgrade sequence as file 05 (upgrade `kubeadm` binary → apply the control-plane upgrade → upgrade the `kubelet` package). *(In this lab's exact wording the `systemctl restart kubelet` step isn't spelled out as its own numbered question, but is implicitly still required in practice for the new kubelet binary to actually take effect.)*

10. **"Run the command `kubectl uncordon master`."**
    ```bash
    kubectl uncordon master
    ```
    - Master is now upgraded and returned to schedulable state.

11. **"Run the command `kubectl drain node01 --ignore-daemonsets`."**
    ```bash
    kubectl drain node01 --ignore-daemonsets
    ```
    - Begin the same rolling procedure on the first worker node.

12. **"Run the commands: `apt install kubeadm=1.18.0-00` and then `kubeadm upgrade node`. Finally, run `apt install kubelet=1.18.0-00`."**
    ```bash
    apt install kubeadm=1.18.0-00
    kubeadm upgrade node
    apt install kubelet=1.18.0-00
    ```
    - Note the worker-node command is **`kubeadm upgrade node`** (not `kubeadm upgrade apply`, which is master/control-plane-only, and also distinct from the file-05 phrasing `kubeadm upgrade node config --kubelet-version vX.Y.Z` — both refer to the same underlying worker-node config-update step, just shown with slightly different flags/wording across the two lessons).

13. **"Run the command `kubectl uncordon node01`."**
    ```bash
    kubectl uncordon node01
    ```
    - Completes the rolling upgrade of `node01`; the same drain → upgrade → uncordon cycle would then be repeated for every remaining worker node.

**Exam-relevant takeaway:** the master is upgraded with `kubeadm upgrade apply vX.Y.Z`; every other (worker) node is upgraded with `kubeadm upgrade node`. Both node types get their `kubeadm` and `kubelet` **packages** upgraded via the OS package manager (`apt install <pkg>=<version>-00`) — `kubeadm` itself never upgrades `kubelet`'s binary, only orchestrates the surrounding cluster-config changes. Always drain before, uncordon after, for every node, including the master.

---

*Continued in [kubernetes-cluster-maintenance-notes_part_02.md](kubernetes-cluster-maintenance-notes_part_02.md)
— Sections 07–11 (Backup and Restore Methods, Working with ETCDCTL, Backup and Restore
Practice Tests x2, Presentation Deck) plus the Quick Revision Checklist.*
