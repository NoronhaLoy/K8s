# Cluster Maintenance — Complete Notes (Part 2 of 2)

> **Part 2 of 2** — covers Sections 07–11 (Backup and Restore Methods, Working with
> ETCDCTL, Backup and Restore Practice Tests x2, Presentation Deck) plus the Quick
> Revision Checklist.
> Previous: [kubernetes-cluster-maintenance-notes_part_01.md](kubernetes-cluster-maintenance-notes_part_01.md)
> (Sections 01–06).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/06-Cluster-Maintenance/`

---

## 07. Backup and Restore Methods

- Video reference: https://kodekloud.com/topic/backup-and-restore-methods/

### Backup Candidates

- **Inferred context (image `bc.PNG`):** likely a two-item summary of *what* you can back up in a Kubernetes cluster, directly matching the two subsections that follow in this same file:
  1. **Resource configuration** (the Kubernetes object manifests/definitions themselves — Deployments, Services, ConfigMaps, etc.).
  2. **The etcd cluster** (the entire cluster state as stored by the control plane's data store).

### Resource Configuration

- **Imperative way**
  - **Inferred context (image `rci.PNG`):** likely illustrates the risk of relying on imperatively-created objects (e.g. `kubectl run`, `kubectl create deployment ...` typed ad hoc at the command line) — if you only ever create resources this way, there is **no manifest file anywhere** recording what you did, so nothing exists to "back up" except the live cluster state itself; if that's lost, the exact object definitions are lost too.
- **Declarative way (Preferred approach)**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    labels:
      app: myapp
      type: front-end
  spec:
    containers:
    - name: nginx-container
      image: nginx
  ```
  - **Inferred context (image `rcd.PNG`):** likely shows this same YAML file being applied with `kubectl apply -f`/`kubectl create -f`, emphasizing that the manifest file on disk **is itself the backup** — you can always recreate the object from the file even if the live cluster object is deleted.
  - **Inferred context (image `rcd1.PNG`):** likely a diagram reinforcing "a good practice is to store resource configurations on source code repositories like GitHub" — showing YAML files committed to a Git repo, from which they can be re-applied to rebuild a cluster's objects at any time (an early "GitOps"-flavored recommendation).

### Backup — Resource Configs

- To take an ad-hoc dump of currently-live resource configs across all namespaces (for a subset of resource types):
  ```
  $ kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml (only for few resource groups)
  ```
  - The parenthetical caveat is important: `kubectl get all` does **not** actually capture every resource type in the cluster (e.g. it typically misses ConfigMaps, Secrets, and various CRDs/other resource groups) — it's a quick approximation, not a comprehensive backup.
- There are many other resource groups that must be considered beyond what `kubectl get all` captures. There are dedicated tools for this, such as:
  - **`ARK`**, now renamed **`Velero`** (by Heptio) — a purpose-built Kubernetes backup/restore tool that can capture cluster resource state (and optionally persistent volume data) far more completely than an ad-hoc `kubectl get all` dump.
  - **Inferred context (image `brc.PNG`):** likely shows Velero/ARK's role visually — cluster resources on one side, an arrow through the Velero/ARK tool, into a backup storage location (e.g. cloud object storage), with a corresponding restore arrow going back into a cluster.

### Backup — ETCD

- Instead of (or in addition to) backing up individual resource configs, you may choose to back up the **etcd cluster itself** — since etcd holds the entire state of the Kubernetes cluster, backing it up captures everything at once.
  - **Inferred context (image `be.PNG`):** likely a simple architecture diagram showing the kube-apiserver as the only component that talks directly to etcd, and etcd as the single source of truth for all cluster objects — visually justifying why "backup etcd" is equivalent to "backup the whole cluster's state."
- You can take a snapshot of the etcd database using the **`etcdctl`** utility's **`snapshot save`** command:
  ```
  $ ETCDCTL_API=3 etcdctl snapshot save snapshot.db
  ```
  ```
  $ ETCDCTL_API=3 etcdctl snapshot status snapshot.db
  ```
  - `ETCDCTL_API=3` is required because `etcdctl` defaults to an older API version on many installs; the v3 API is what supports `snapshot save`/`snapshot restore`.
  - `snapshot status` reports metadata about a snapshot file (e.g. hash, revision number, total keys, total size) without restoring it — useful to sanity-check a backup file.
  - **Inferred context (image `be1.PNG`):** likely shows sample output of both commands — `snapshot save` printing a confirmation line with the snapshot's byte size, and `snapshot status` printing a small table with columns like `hash | revision | total keys | total size`.

### Restore — ETCD

- To restore etcd from a backup taken earlier:
  1. **First, stop the `kube-apiserver` service** (since it's the only component talking to etcd, and etcd's data directory is about to be replaced/pointed elsewhere):
     ```
     $ service kube-apiserver stop
     ```
  2. **Run the `etcdctl snapshot restore` command** (restores the snapshot file into a fresh data directory).
  3. **Update the etcd service** configuration to point at that restored data directory.
  4. **Reload systemd's configuration:**
     ```
     $ systemctl daemon-reload
     ```
  5. **Restart etcd:**
     ```
     $ service etcd restart
     ```
     - **Inferred context (image `er.PNG`):** likely a flow diagram of exactly this 5-step sequence — stop `kube-apiserver` → `etcdctl snapshot restore` → edit etcd's config/data-dir → `daemon-reload` → restart `etcd`.
  6. **Start the `kube-apiserver`** again once etcd is back up on the restored data:
     ```
     $ service kube-apiserver start
     ```
- **With all `etcdctl` commands, you must specify the cert, key, cacert, and endpoint for authentication** (etcd's client API is secured with mutual TLS in a typical kubeadm cluster):
  ```
  $ ETCDCTL_API=3 etcdctl \
    snapshot save /tmp/snapshot.db \
    --endpoints=https://[127.0.0.1]:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/etcd-server.crt \
    --key=/etc/kubernetes/pki/etcd/etcd-server.key
  ```
  - **Inferred context (image `erest.PNG`):** likely shows this same flag set applied to a `snapshot restore` invocation as well (or reiterates that these four flags — `--endpoints`, `--cacert`, `--cert`, `--key` — must accompany essentially every `etcdctl` operation against a live, TLS-secured etcd endpoint), and/or shows the default kubeadm certificate file paths under `/etc/kubernetes/pki/etcd/` referenced by these flags.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/

---

## 08. Working with ETCDCTL

- Video reference: *Working with ETCDCTL* — https://kodekloud.com/topic/working-with-etcdctl/
- **This source file itself contains no inline written content beyond the title and the video link** — it is a placeholder page pointing to a hands-on demo video, with no accompanying text/YAML/commands captured in the docs export.
- **Inferred context (video: "Working with ETCDCTL") — likely content, based on the immediately preceding lesson (file 07) and general `etcdctl` usage this demo would naturally cover:**
  - Confirming which `etcdctl` API version is active and forcing v3 behavior:
    ```
    $ ETCDCTL_API=3 etcdctl version
    ```
  - Locating the etcd certificate/key files used for authentication on a kubeadm-managed control-plane node (typically under `/etc/kubernetes/pki/etcd/`), and identifying the etcd client endpoint (typically `https://127.0.0.1:2379`) — most easily read from the running `etcd` static Pod's spec:
    ```
    $ kubectl describe pod etcd-controlplane -n kube-system
    ```
  - Demonstrating a full `snapshot save` with all four authentication flags together (as introduced in file 07):
    ```
    $ ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot.db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key
    ```
  - Verifying the resulting backup file with `snapshot status`, then performing a `snapshot restore --data-dir <new-path>` and walking through updating the static Pod manifest's `hostPath` for the etcd data volume, exactly as codified later in the Practice Test solutions (files 09 and 10) — this file most likely served as the live-demo companion to that written lab, without its own separate transcript captured here.
- **Because there is no original written text to quote beyond the title/link, no verbatim commands from this specific file can be preserved** — all `etcdctl` command syntax for this section is captured verbatim in files 07, 09, and 10 instead.

---

## 09. Practice Test — Backup and Restore Methods

Solutions to practice test — Backup and Restore Methods. Step-by-step lab-guide walkthrough:

1. **"How many deployments exist in the cluster?"**
   ```
   kubectl get deployments
   ```
   - Count the rows returned.

2. **"What is the version of ETCD running on the cluster?"**
   ```
   kubectl describe pod -n kube-system etcd-controlplane
   ```
   - Find the entry for **`Image`** in the description output (the image tag encodes the etcd version, e.g. `registry.k8s.io/etcd:3.5.x-0`).

3. **"At what address can you reach the ETCD cluster from the controlplane node?"**
   ```
   kubectl describe pod -n kube-system etcd-controlplane
   ```
   - Under **`Command`**, find the **`--listen-client-urls`** flag — its value(s) are the address(es) etcd is listening on for client connections (typically `https://127.0.0.1:2379` and/or a node-IP-based URL).

4. **"Where is the ETCD server certificate file located?"**
   - On kubeadm clusters like this one, the **default location for certificate files is `/etc/kubernetes/pki/etcd`**.
   - Task: choose the correct certificate file from that directory (the server certificate, typically named `server.crt`, as opposed to the CA cert, peer certs, etc.).

5. **"Where is the ETCD CA Certificate file located?"**
   - Same directory, **`/etc/kubernetes/pki/etcd`** — task: choose the correct certificate (the CA cert, typically `ca.crt`, as opposed to the server cert used in the previous question).

6. **"Take a snapshot of the ETCD database using the built-in snapshot functionality. Store the backup file at location `/opt/snapshot-pre-boot.db`."**
   ```
   ETCDCTL_API=3 etcdctl snapshot save \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key \
     /opt/snapshot-pre-boot.db
   ```
   - Note: no `--endpoints` flag is shown here explicitly — `etcdctl` defaults to `127.0.0.1:2379` when run locally on the control-plane node, so it can be omitted when running directly on that node.

7. **(Information-only step — no command/question, purely narrative framing for the disaster scenario that follows.)**

8. **"Wake up! We have a conference call! After the reboot the master nodes came back online, but none of our applications are accessible. Check the status of the applications on the cluster. What's wrong?"**
   - **Answer:** *All of the above* — i.e. the practice test presents this as a multiple-choice question where every listed symptom (e.g. Deployments missing, Pods missing, Services missing, etc.) is simultaneously true, because the cluster's etcd state was lost/corrupted across the reboot, wiping out **all** cluster object records at once (not just one resource type).

9. **"Luckily we took a backup. Restore the original state of the cluster using the backup file."**
   - **Step 1 — Restore the backup to a new directory:**
     ```
     ETCDCTL_API=3 etcdctl snapshot restore \
       --data-dir /var/lib/etcd-from-backup \
       /opt/snapshot-pre-boot.db
     ```
     - This creates a **brand-new** etcd data directory (`/var/lib/etcd-from-backup`) populated from the snapshot — it does **not** touch or overwrite the currently-configured/broken etcd data directory in place.
   - **Step 2 — Modify the `etcd` static Pod manifest to use the new directory:**
     - Edit the `volumes` section and change the `hostPath` so the `etcd-data` volume points at the newly restored directory:
       ```
       vi /etc/kubernetes/manifests/etcd.yaml
       ```
       ```yaml
         volumes:
         - hostPath:
             path: /etc/kubernetes/pki/etcd
             type: DirectoryOrCreate
           name: etcd-certs
         - hostPath:
             path: /var/lib/etcd      # <- change this
             type: DirectoryOrCreate
           name: etcd-data
       ```
       - New value for the `etcd-data` volume's `hostPath.path`: **`/var/lib/etcd-from-backup`**.
     - **Save the file and wait up to a minute** for the `etcd` static Pod to reload — because this is a **static Pod** manifest under `/etc/kubernetes/manifests/`, the kubelet on that node watches this directory and automatically restarts the Pod when the file changes; there is no need to `kubectl delete`/`kubectl apply` it manually, and no need to restart the kubelet.
   - **Step 3 — Verify:**
     ```
     kubectl get deployments
     kubectl get services
     ```
     - Confirms the previously "missing" Deployments/Services have reappeared, proving the restore repopulated etcd (and therefore the whole API-visible cluster state) correctly.

**See also:** https://github.com/kodekloudhub/community-faq/blob/main/docs/etcd-faq.md

**Exam-relevant takeaway:** `etcdctl snapshot restore --data-dir <new-path>` always creates a fresh directory — you then point the etcd **static Pod manifest's** `hostPath` at that new directory (never edit the live/old data directory in place). Because etcd runs as a static Pod, editing the manifest file is enough; the kubelet reconciles it automatically.

---

## 10. Practice Test — Backup and Restore Methods 2

Solutions to practice test — Backup and Restore Methods 2. This test practices with **both** *stacked* etcd (etcd running as a Pod alongside the control plane, as in the previous test) **and** *external* etcd (etcd running as its own separate OS service on separate node(s), not as a Kubernetes Pod at all).

1. **(Information-only step — introduces the dual-cluster, stacked-vs-external scenario.)**

2. **"Explore the student-node and the clusters it has access to."**
   ```bash
   kubectl config get-contexts
   ```

3. **"How many clusters are defined in the kubeconfig on the student-node?"**
   ```bash
   kubectl config get-contexts
   ```
   - **Answer: 2** (`cluster1` and `cluster2`).

4. **"How many nodes (both controlplane and worker) are part of cluster1?"**
   ```bash
   kubectl config use-context cluster1
   kubectl get nodes
   ```
   - **Answer: 2.**

5. **"What is the name of the controlplane node in cluster2?"**
   ```bash
   kubectl config use-context cluster2
   kubectl get nodes
   ```
   - **Answer: `cluster2-controlplane`.**

6. **(Information-only step.)**

7. **"How is ETCD configured for cluster1?"**
   ```bash
   kubectl config use-context cluster1
   kubectl get pods -n kube-system
   ```
   - **Answer: `Stacked ETCD`** — the output shows a Pod for etcd (e.g. `etcd-cluster1-controlplane`) running in `kube-system`, meaning etcd is co-located with (stacked on) the control-plane node as a static Pod.

8. **"How is ETCD configured for cluster2?"**
   ```bash
   kubectl config use-context cluster2
   kubectl get pods -n kube-system
   ```
   - **Answer: `External ETCD`** — no etcd Pod appears in the output at all. Since a functioning cluster cannot exist with no etcd whatsoever, the only remaining explanation is that etcd is running **externally**, outside of Kubernetes entirely (as an OS-level service on a separate node), rather than as a Pod.

9. **"What is the IP address of the External ETCD datastore used in cluster2?"**
   ```bash
   kubectl config use-context cluster2
   kubectl get pods -n kube-system kube-apiserver-cluster2-controlplane -o yaml | grep etcd
   ```
   - Locate the **`--etcd-servers`** flag in the kube-apiserver Pod's spec/command — its value is the connection string (IP + port) the API server uses to reach the external etcd datastore. The IP address in that line is the answer.

10. **"What is the default data directory used for the ETCD datastore used in cluster1?"**
    - Examine the etcd static Pod manifest on the control-plane node and find the `hostPath` of its `etcd-data` volume:
      ```bash
      kubectl config use-context cluster1
      kubectl get pods -n kube-system etcd-cluster1-controlplane -o yaml
      ```
    - In the `volumes` section, the host path of the volume named `etcd-data` is the answer.
    - **Answer: `/var/lib/etcd`.**

11. **(Information-only step.)**

12. **"What is the default data directory used for the ETCD datastore used in cluster2?"**
    - Since cluster2 uses **external** etcd, there is no Pod/manifest to inspect — instead, examine the **systemd unit file** for the etcd OS service directly:
      ```bash
      ssh etcd-server
      ```
      ```bash
      # Verify the name of the service
      systemctl list-unit-files | grep etcd

      # Using the output from the above command
      systemctl cat etcd.service
      ```
    - Note the comment line in the output, which tells you the location of the actual service unit file on disk — this location will be needed again in a later step when the file must be edited.
    - From the output, locate the **`--data-dir`** flag.
    - **Answer: `/var/lib/etcd-data`.**
    - Return to the student node:
      ```bash
      exit
      ```

13. **"How many other nodes are part of the ETCD cluster that etcd-server is a part of?"**
    - The source notes this question is somewhat contentious/ambiguously worded (it "ought not to contain the word `other`").
    - **Required answer: 1.**

14. **"Take a backup of etcd on cluster1 and save it on the student-node at the path `/opt/cluster1.db`."**
    - Since cluster1 is **stacked** etcd, the backup must be performed **on the control-plane node**, then pulled back to the student-node:
      ```bash
      ssh cluster1-controlplane
      ```
      ```bash
      ETCDCTL_API=3 etcdctl snapshot save \
        --cacert /etc/kubernetes/pki/etcd/ca.crt \
        --cert /etc/kubernetes/pki/etcd/server.crt \
        --key /etc/kubernetes/pki/etcd/server.key \
        cluster1.db

      # Return to student node
      exit
      ```
      ```bash
      scp cluster1-controlplane:~/cluster1.db /opt/
      ```

15. **"An ETCD backup for cluster2 is stored at `/opt/cluster2.db`. Use this snapshot file to carry out a restore on cluster2 to a new path `/var/lib/etcd-data-new`."**
    - As established earlier, `cluster2` uses **external** etcd, which means:
      - `etcd` does **not** have to live on the control-plane node of the cluster — and indeed, in this scenario, it does not (it runs on the separate `etcd-server` node).
      - `etcd` runs as an **operating system service, not a Pod** — therefore there is **no manifest file to edit**; instead, changes must be made to a **systemd service unit file**.
    - This question has several sub-parts:
      1. **Move the backup to the etcd-server node:**
         ```bash
         scp /opt/cluster2.db etcd-server:~/
         ```
      2. **Log into the etcd-server node:**
         ```bash
         ssh etcd-server
         ```
      3. **Check the ownership of the current etcd-data directory** — required because the restored data's ownership must match what etcd expects; the data directory's location was determined back in step 12:
         ```bash
         ls -ld /var/lib/etcd-data/
         ```
         - Note: owner and group are both **`etcd`**.
      4. **Do the restore:**
         ```bash
         ETCDCTL_API=3 etcdctl snapshot restore \
             --data-dir /var/lib/etcd-data-new \
             cluster2.db
         ```
      5. **Set ownership on the restored directory** (to match what was observed in the ownership check above):
         ```bash
         chown -R etcd:etcd /var/lib/etcd-data-new
         ```
      6. **Reconfigure and restart etcd** — requires the location of the service unit file, also determined in step 12:
         ```bash
         vi /etc/systemd/system/etcd.service
         ```
         - Edit the **`--data-dir`** argument to point at the newly restored directory (`/var/lib/etcd-data-new`), then save.
         - Because a service unit file was edited, a `daemon-reload` is required to reload systemd's in-memory configuration before restarting the service:
           ```bash
           systemctl daemon-reload
           systemctl restart etcd.service
           ```
         - Return to the student node:
           ```bash
           exit
           ```
      7. **Verify the restore:**
         ```bash
         kubectl config use-context cluster2
         kubectl get all -n critical
         ```

**See also:** https://github.com/kodekloudhub/community-faq/blob/main/docs/etcd-faq.md

**Exam-relevant takeaway:** the core `etcdctl snapshot save`/`snapshot restore` commands are **identical** regardless of stacked vs external etcd — what differs is *where* you run them (control-plane node for stacked, the dedicated etcd node for external) and *how* you reconfigure etcd to use the restored data afterward: edit a **static Pod manifest** (`/etc/kubernetes/manifests/etcd.yaml`) for stacked etcd (auto-reloaded by kubelet), versus edit a **systemd unit file** (e.g. `/etc/systemd/system/etcd.service`) plus `systemctl daemon-reload` + `systemctl restart etcd.service` for external etcd. Always re-check/fix directory **ownership** (`etcd:etcd`) after restoring into a new external data directory, since `etcdctl snapshot restore` creates the new directory but doesn't necessarily preserve the exact ownership etcd expects.

---

## 11. Download Presentation Deck

- The section provides a downloadable slide deck accompanying the video lectures: [Presentation Deck](https://kodekloud.com/topic/download-presentation-deck-5/).
- Purely a resource/reference link — no technical content of its own (identical in nature/format to the equivalent "Download Presentation Deck" page at the end of the Application Lifecycle Management section).

---

## Quick Revision Checklist

- [ ] **Node maintenance: cordon / drain / uncordon**
  - `kubectl cordon <node>` — marks the node **unschedulable only**; existing Pods are left running untouched.
  - `kubectl drain <node>` — cordons **and** safely evicts existing Pods so a controller (ReplicaSet/Deployment/etc.) reschedules them elsewhere.
    - `--ignore-daemonsets` — required whenever DaemonSet Pods are present (they can't be "moved," so drain would otherwise refuse).
    - `--force` — required to evict a **bare Pod** not backed by any controller; doing so **permanently deletes** that Pod (nothing recreates it) — a classic exam trap ("what happens to `hr-app`?").
  - `kubectl uncordon <node>` — makes the node schedulable again, but does **not** retroactively move any Pods back onto it; only future scheduling decisions are affected.
  - **Unplanned node loss:** if a node stays `NotReady`/unreachable for longer than the default **`pod-eviction-timeout`** (5 minutes), its Pods are evicted/rescheduled automatically by the control plane.

- [ ] **PodDisruptionBudget (PDB) interaction**
  - `kubectl drain` respects any configured **PodDisruptionBudget** for the Pods it's evicting — it will **not** evict a Pod if doing so would violate the PDB's `minAvailable`/`maxUnavailable` constraints, and the drain can stall/retry until it's safe (general Kubernetes knowledge complementing this section's drain coverage — not explicit in the source text itself, but directly relevant to how `drain` behaves in real/exam clusters).

- [ ] **Version skew policy**
  - Kubernetes components are **not required to be on identical versions**.
  - Only the most **recent 3 minor versions** are supported at any given time.
  - Always **upgrade one minor version at a time** (never skip a minor version).
  - `kubectl get nodes`'s `VERSION` column reports each node's **kubelet** version, not the API server's version.

- [ ] **kubeadm cluster upgrade workflow**
  - Two major phases: **(1) upgrade the master/control-plane**, **(2) upgrade each worker node.**
  - **Master:**
    1. `kubeadm upgrade plan` — check what's available/compatible.
    2. `apt-get upgrade -y kubeadm=<version>-00` — upgrade the `kubeadm` tool itself.
    3. `kubeadm upgrade apply v<version>` — upgrades the control-plane components.
    4. `apt-get upgrade kubelet=<version>-00` then `systemctl restart kubelet` — upgrade + restart the master's own kubelet.
    5. Drain the master first (`kubectl drain master --ignore-daemonsets`) and uncordon it after (`kubectl uncordon master`).
  - **Each worker node (one at a time, to avoid downtime):**
    1. `kubectl drain <node> --ignore-daemonsets` (run from the master/control-plane).
    2. `apt-get upgrade -y kubeadm=<version>-00` and `apt-get upgrade -y kubelet=<version>-00` on that node.
    3. `kubeadm upgrade node` (or, as also phrased in the lectures, `kubeadm upgrade node config --kubelet-version v<version>`) to update the node's local config for the new kubelet.
    4. `systemctl restart kubelet`.
    5. `kubectl uncordon <node>`.
  - **Worker upgrade strategies:** all-at-once (downtime, simplest), one-node-at-a-time (no downtime, no new infra needed — the default exam answer when "no new VMs" + "no user impact" are both required), or add-new-nodes-then-retire-old (needs extra infra, no downtime).

- [ ] **etcd backup with `etcdctl`**
  - Always prefix with `ETCDCTL_API=3` for snapshot commands.
  - **Backup:**
    ```
    ETCDCTL_API=3 etcdctl snapshot save <path-to-file>.db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key
    ```
  - Verify with `etcdctl snapshot status <file>.db`.
  - Every authenticated `etcdctl` call needs the same 4 things: `--endpoints`, `--cacert`, `--cert`, `--key` — default kubeadm cert location is `/etc/kubernetes/pki/etcd/`.

- [ ] **etcd restore and `--data-dir`**
  - `etcdctl snapshot restore --data-dir <new-empty-path> <file>.db` — **always** restores into a brand-new directory; never point it at the currently-in-use data directory.
  - **Stacked etcd (etcd as a static Pod):** after restoring, edit `/etc/kubernetes/manifests/etcd.yaml`'s `volumes` section, changing the `etcd-data` volume's `hostPath.path` to the new restored directory. Because it's a **static Pod manifest**, the kubelet auto-detects the change and recreates the Pod within about a minute — no manual `kubectl delete`/`apply`, no kubelet restart needed.
  - **External etcd (etcd as a systemd service):** no manifest exists; instead edit the **systemd unit file** (e.g. `/etc/systemd/system/etcd.service`), change its `--data-dir` argument to the new directory, then run `systemctl daemon-reload` followed by `systemctl restart etcd.service`.
  - Also remember to **fix ownership** (`chown -R etcd:etcd <new-data-dir>`) on a freshly-restored external etcd data directory before restarting the service, matching the ownership of the original data directory.
  - Full sequence for stacked etcd, spelled out: stop `kube-apiserver` → `etcdctl snapshot restore` → point etcd config/manifest at new data dir → `systemctl daemon-reload` → restart `etcd` → start `kube-apiserver` again.
  - Stacked vs external etcd is distinguished by whether `kubectl get pods -n kube-system` shows an `etcd-*` Pod: present = stacked; absent = external (find the external etcd IP via the kube-apiserver Pod's `--etcd-servers` flag).

- [ ] **General exam habits reinforced across this section**
  - `kubectl describe pod <etcd-pod> -n kube-system` is the go-to command to read etcd's image/version, `--listen-client-urls`, and cert flag paths on a stacked-etcd cluster.
  - Deployments/Services/etc. disappearing after a reboot is a classic symptom pointing at **etcd data loss**, restorable only via a prior etcd snapshot.
  - Resource-config backups (`kubectl get all --all-namespaces -o yaml`, or better, tools like **Velero**/ARK) are a *complementary*, not equivalent, backup strategy to backing up etcd directly — `kubectl get all` misses many resource groups.
  - Declarative YAML manifests kept in source control (e.g. GitHub) are the preferred way to make resource configuration inherently "backed up" as a side effect of normal workflow.

---

*End of Cluster Maintenance notes.*
