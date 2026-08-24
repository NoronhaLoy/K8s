# Kubernetes Core Concepts — Complete Notes (with Worked Examples)

> Covers the full **02-Core-Concepts** module, topics 01–25, end to end: Section
> Introduction, Cluster Architecture, Docker vs ContainerD, ETCD (standalone + in
> Kubernetes), Kube-API Server, Kube-Controller-Manager, Kube-Scheduler, Kubelet,
> Kube-Proxy, Pods, ReplicaSets, Deployments, Namespaces, Services (ClusterIP /
> NodePort / LoadBalancer), Imperative Commands with kubectl, and Attachments — plus
> every "Practice Test" lecture, folded into the topic it drills.
>
> Every topic follows the same shape: **🎯 one-sentence summary → Concept (plain
> language) → Diagram → YAML/Commands → Worked Example (step by step) → ⚠️ Exam Tip →
> Key Takeaways / Traps.** Use the Table of Contents or the Cheat Sheet below to jump
> straight to what you need.

---

<a id="toc"></a>
## Table of Contents

- [Big Picture — How All the Pieces Fit Together](#big-picture)
- [Quick-Reference Cheat Sheet](#cheat-sheet)
- [01. Core Concepts — Section Introduction](#t01)
- [02. Cluster Architecture](#t02)
- [03. Docker vs ContainerD](#t03)
- [04. ETCD for Beginners](#t04)
- [05. ETCD in Kubernetes](#t05)
- [06. Kube-API Server](#t06)
- [07. Kube-Controller-Manager](#t07)
- [08. Kube-Scheduler](#t08)
- [09. Kubelet](#t09)
- [10. Kube-Proxy](#t10)
- [11. Pods](#t11)
- [12. Practice Test Introduction](#t12)
- [13. Practice Test - PODs](#t13)
- [14. ReplicaSets](#t14)
- [15. Practice Tests - ReplicaSet](#t15)
- [16. Deployments](#t16)
- [17. Practice Tests - Deployments](#t17)
- [18. Namespaces](#t18)
- [19. Practice Test - Namespaces](#t19)
- [20. Services](#t20)
- [21. Services - ClusterIP](#t21)
- [22. Practice Test - Services](#t22)
- [23. Imperative Commands with kubectl](#t23)
- [24. Practice Test - Imperative Commands](#t24)
- [25. Attachments](#t25)

---

<a id="cheat-sheet"></a>
## Quick-Reference Cheat Sheet

One row per topic — read top to bottom for a 2-minute refresher before an exam/lab.

| # | Topic | One-liner | Key command(s) | #1 thing to watch out for |
|---|---|---|---|---|
| 01 | Section Introduction | Roadmap: ETCD → API server → Scheduler/Controller-Manager → Kubelet → Kube-Proxy → the objects you author (Pod→RS→Deploy→NS→Svc) | `kubectl get pods -n kube-system` | Orientation only — no gotchas yet |
| 02 | Cluster Architecture | Control plane makes decisions & stores state; worker nodes run the actual workloads | `kubectl get nodes`, `kubectl get pods -n kube-system -o wide` | **kubelet is NOT a pod**; **kube-proxy IS a pod** |
| 03 | Docker vs ContainerD | K8s never runs containers itself — it always delegates via CRI; Docker's dockershim is gone (removed 1.24), containerd/CRI-O talk CRI directly | `crictl ps`, `crictl images` | Use `crictl`, not `docker`, on modern nodes |
| 04 | ETCD for Beginners | Generic distributed key-value store using Raft consensus; nothing K8s-specific about it | `etcdctl put/get`, `export ETCDCTL_API=3` | Forgetting `ETCDCTL_API=3` mixes v2/v3 syntax |
| 05 | ETCD in Kubernetes | Stores every cluster object under `/registry/...`; only `kube-apiserver` ever touches it directly | `etcdctl snapshot save/restore`, `etcdctl member list` | Restore into a **new** `--data-dir`, never in-place |
| 06 | Kube-API Server | The one front door for every request: Authenticate → Authorize → Admission → write etcd → notify watchers | `kubectl auth can-i`, read `kube-apiserver.yaml` | `--authorization-mode` order matters (`Node,RBAC`) |
| 07 | Kube-Controller-Manager | One process bundling many control loops (Node, ReplicaSet, Endpoints, Namespace controllers...) | `kubectl describe node`, check `--leader-elect` | grace-period (40s) vs eviction-timeout (5m) are separate flags |
| 08 | Kube-Scheduler | Filter → Score pipeline decides which node an unscheduled Pod lands on | `kubectl get events --field-selector reason=Scheduled` | No `Scheduled` event at all → scheduler itself isn't running |
| 09 | Kubelet | The only non-pod control-plane-adjacent agent — must already be running to create the other static pods | `systemctl status kubelet`, `journalctl -u kubelet` | Node `NotReady` → check kubelet **on that node**, not `kubectl` |
| 10 | Kube-Proxy | DaemonSet that writes iptables/IPVS rules so Service traffic reaches the right Pods | `kubectl get endpoints`, `iptables-save \| grep KUBE-SERVICES` | Empty `Endpoints` = bad Service `selector`, **not** a kube-proxy bug |
| 11 | Pods | Smallest deployable unit — every container is always wrapped in a Pod, even a lone one | `kubectl run`, `kubectl describe pod`, `kubectl logs -c <container>` | `READY` = containers-ready/total **for one Pod**, not a pod count |
| 12 | Practice Test Introduction | Meta/orientation lecture — no K8s content | — | — |
| 13 | Practice Test - PODs | Hands-on drill of topic 11 | `kubectl get pods --no-headers \| wc -l` | — |
| 14 | ReplicaSets | Controller that keeps N identical Pod replicas alive — this *is* self-healing | `kubectl scale`, `kubectl get rs` | Editing a running RS's `image` does **not** touch already-running Pods |
| 15 | Practice Tests - ReplicaSet | Hands-on drill of topic 14 | `kubectl explain rs \| grep VERSION` | — |
| 16 | Deployments | Adds rollout history + rolling update/rollback on top of a ReplicaSet | `kubectl rollout status/history/undo`, `kubectl set image` | Old ReplicaSet stays at **0** replicas on purpose — that's rollback history, not a leftover bug |
| 17 | Practice Tests - Deployments | Hands-on drill of topic 16 | `kubectl create deployment --image= --replicas=` | — |
| 18 | Namespaces | Virtual sub-clusters for team/project isolation; a few resource types are cluster-scoped and never namespaced | `kubectl get pods --all-namespaces`, `kubectl config set-context --current --namespace=` | Nodes / PVs / StorageClasses / ClusterRoles are **never** namespaced |
| 19 | Practice Test - Namespaces | Hands-on drill of topic 18 | `kubectl get pods -n <ns>` | — |
| 20 | Services | Stable virtual IP + DNS name in front of a changing set of Pods (ClusterIP / NodePort / LoadBalancer) | `kubectl expose`, `kubectl describe service` | `port` vs `targetPort` vs `nodePort` mix-up; `nodePort` must be 30000–32767 |
| 21 | Services - ClusterIP | Internal-only tier-to-tier traffic; the default Service type | `kubectl describe service \| grep Endpoints` | Empty `Endpoints` = selector/label mismatch, not a kube-proxy issue |
| 22 | Practice Test - Services | Hands-on drill of topics 20–21 | `kubectl describe service \| grep TargetPort` | — |
| 23 | Imperative Commands with kubectl | Generate YAML fast with `--dry-run=client -o yaml` instead of hand-writing it | `kubectl run/create ... --dry-run=client -o yaml`, `kubectl expose` | `kubectl create service` defaults `selector` to `app=<name>` — may not match real Pods |
| 24 | Practice Test - Imperative Commands | Hands-on drill of topic 23 | `kubectl run ... --expose --port=` | — |
| 25 | Attachments | Just links to the course's presentation decks — no technical content | — | — |

---

<a id="big-picture"></a>
## Big Picture — How All the Pieces Fit Together

> 🎯 **In one sentence:** every request flows through `kube-apiserver`, which is the
> only component allowed to talk to `etcd` — nothing else in the diagram below ever
> bypasses it.

```
                              KUBERNETES CLUSTER
   ┌───────────────────────────────────────────────────────────────────────┐
   │                         CONTROL PLANE NODE(S)                          │
   │                                                                         │
   │   ┌───────────┐   ┌────────────┐   ┌───────────────────┐   ┌─────────┐ │
   │   │  ETCD     │◄──│kube-       │──►│kube-controller-    │   │ kube-   │ │
   │   │ (cluster  │   │apiserver   │   │manager             │   │scheduler│ │
   │   │  state)   │   │(front door)│   │(watches & reconciles│  │(assigns │ │
   │   └───────────┘   └─────┬──────┘   │ desired state)     │   │ pods to │ │
   │                         │           └───────────────────┘   │ nodes)  │ │
   │                         │                                    └────┬────┘ │
   └─────────────────────────┼──────────────────────────────────────────┼─────┘
                              │  all components talk ONLY to apiserver, │
                              │  never directly to each other or etcd   │
                              ▼                                          ▼
   ┌───────────────────────────────────────────────────────────────────────┐
   │                            WORKER NODE(S)                              │
   │   ┌───────────┐        ┌───────────┐        ┌─────────────────────┐   │
   │   │  kubelet  │◄──────►│kube-proxy │        │ Container Runtime    │   │
   │   │ (talks to │        │(networking│◄──────►│ (containerd/CRI-O,   │   │
   │   │ apiserver,│        │ rules,    │        │  via CRI) runs the   │   │
   │   │ manages   │        │ services) │        │  actual containers   │   │
   │   │ pods here)│        └───────────┘        └─────────────────────┘   │
   │   └───────────┘                                                       │
   └───────────────────────────────────────────────────────────────────────┘

   Golden rule: the API server is the ONLY component that reads/writes etcd, and the
   ONLY component every other piece (scheduler, controller-manager, kubelet, kubectl,
   even kube-proxy) talks to. Nothing bypasses it.
```

---

<a id="t01"></a>
## 01. Core Concepts — Section Introduction

> 🎯 **In one sentence:** this section builds the cluster's mental model from the
> ground up — data store → control plane → worker agents → the objects you actually
> create — in the exact order a request touches them.

**Concept (in plain terms):** Kubernetes has a lot of moving parts, so this section
introduces them in **dependency order**, not alphabetical order — each new topic only
makes sense once you understand the one before it.

| Step | Topic | Why it comes at this point |
|---|---|---|
| 1 | **Cluster Architecture** | Names every moving part and which node type it lives on — the map you'll refer back to. |
| 2 | **Docker vs ContainerD** | Kubernetes never runs containers itself — you need to know what actually does, underneath. |
| 3 | **ETCD** (standalone, then in-Kubernetes) | The database that *is* the cluster's brain; almost everything else just reads/writes/watches it. |
| 4 | **Kube-API Server, Controller-Manager, Scheduler** | The three control-plane processes, in the order a Pod's lifecycle actually touches them. |
| 5 | **Kubelet, Kube-Proxy** | The two worker-node agents that make Pods actually run and be reachable. |
| 6 | **Pods → ReplicaSets → Deployments → Namespaces → Services** | The object hierarchy *you* author, building from the smallest unit up to traffic routing. |
| 7 | **Imperative commands with kubectl** | The fast, exam-realistic way to create/edit those objects without hand-writing YAML every time. |

### Diagram — Section roadmap (what depends on what)

```
 ETCD (data store)
   │
   ▼
 kube-apiserver (reads/writes etcd; everything else's front door)
   │
   ├──► kube-scheduler         (decides WHICH node)
   ├──► kube-controller-manager(keeps desired == actual state)
   │
   ▼
 kubelet (on every worker node) ──► talks to Container Runtime (containerd/CRI-O)
   │                                  to actually start/stop containers
   ▼
 kube-proxy (on every worker node) ──► sets up networking rules so Services work
   │
   ▼
 Pod → ReplicaSet → Deployment → Namespace → Service   (what YOU create as a user)
```

### Worked Example — Seeing the whole roadmap on a live cluster in 60 seconds

```bash
# 1. Control-plane components (all run as static pods in kube-system on kubeadm clusters)
kubectl get pods -n kube-system
# NAME                                READY   STATUS
# etcd-controlplane                  1/1     Running
# kube-apiserver-controlplane        1/1     Running
# kube-controller-manager-controlplane 1/1   Running
# kube-scheduler-controlplane        1/1     Running
# kube-proxy-xxxxx                   1/1     Running   (DaemonSet, one per node)
# coredns-xxxxx                      1/1     Running

# 2. Worker-node agent (NOT a pod — runs as a systemd service on the node itself)
ssh node01
service kubelet status
# active (running)

# 3. Container runtime underneath kubelet
crictl info | grep -i version
```

This single `kubectl get pods -n kube-system` command is the fastest way to confirm
every control-plane piece this section is about to explain is actually alive.

> ⚠️ **Exam Tip:** if you only remember one command from this whole section, make it
> `kubectl get pods -n kube-system` — it's the fastest sanity check that every
> control-plane piece (etcd, apiserver, scheduler, controller-manager, kube-proxy) is
> actually alive, before you go troubleshooting anything else.

---

<a id="t02"></a>
## 02. Cluster Architecture

> 🎯 **In one sentence:** a cluster is two node roles — control-plane nodes that
> *decide* things and store state, and worker nodes that *run* things.

**Concept (in plain terms):** A Kubernetes cluster is split into two node roles:

| Role | Runs | Purpose |
|---|---|---|
| **Control Plane (Master) node(s)** | kube-apiserver, etcd, kube-scheduler, kube-controller-manager, (cloud-controller-manager if on a cloud) | Makes *global* decisions (scheduling, detecting/responding to cluster events) and stores all cluster state. |
| **Worker node(s)** | kubelet, kube-proxy, container runtime | Actually run application workloads (Pods) as instructed by the control plane. |

### Control plane components, one-line each

- **kube-apiserver** — the front door; every `kubectl` command, every other
  component, and every user request passes through here. Validates & processes REST
  requests, then persists to etcd.
- **etcd** — distributed, consistent key-value store holding the *entire* cluster
  state (nodes, pods, configs, secrets, roles, everything).
- **kube-scheduler** — watches for Pods with no node assigned, picks the best node
  (Filter → Score, see the Scheduling notes file for full depth).
- **kube-controller-manager** — runs all the built-in *controllers* (Node Controller,
  Replication Controller, Endpoints Controller, Service Account & Token Controllers,
  etc.) — each is a control loop watching apiserver state and reconciling toward
  desired state.
- **cloud-controller-manager** — same idea, but for cloud-provider-specific logic
  (load balancers, volumes, node lifecycle tied to the cloud API) — only present on
  managed/cloud clusters.

### Worker node components, one-line each

- **kubelet** — the agent on every worker node; registers the node with the
  apiserver, and ensures containers described in Pod specs assigned to *this* node are
  actually running and healthy.
- **kube-proxy** — maintains network rules on each node (iptables/IPVS) so that
  Services can route traffic to the right backend Pods.
- **Container runtime** — the actual software that runs containers (containerd,
  CRI-O, etc.) — see next topic.

### Diagram — Full component map with request flow

```
                     kubectl create -f pod.yaml
                                │
                                ▼
                    ┌───────────────────────┐
                    │    kube-apiserver      │  (validates, auths, persists)
                    └───────────┬───────────┘
                                │ writes
                                ▼
                    ┌───────────────────────┐
                    │        etcd            │  (Pod object stored, nodeName empty)
                    └───────────┬───────────┘
                                │ apiserver notifies watchers
                    ┌───────────┴───────────┐
                    ▼                       ▼
        ┌─────────────────────┐   ┌─────────────────────────┐
        │   kube-scheduler     │   │  kube-controller-manager │
        │  sees unscheduled Pod│   │  (not involved for a bare │
        │  picks node02        │   │   Pod; matters for       │
        │  writes Binding via  │   │   ReplicaSet/Deployment) │
        │  apiserver -> etcd   │   └─────────────────────────┘
        └───────────┬───────────┘
                     │ apiserver notifies node02's kubelet
                     ▼
        ┌─────────────────────┐
        │   kubelet (node02)   │  pulls image via container runtime, starts container
        └───────────┬───────────┘
                     ▼
        ┌─────────────────────┐
        │ container runtime    │  (containerd, via CRI)
        │  (node02)             │
        └───────────────────────┘

        Meanwhile, kube-proxy on every node updates iptables/IPVS rules whenever
        Services/Endpoints change, so traffic can reach this new Pod through a Service.
```

### Worked Example — Mapping every component to a real command

```bash
# See both node roles
kubectl get nodes
# NAME            STATUS   ROLES           AGE
# controlplane    Ready    control-plane   10d
# node01          Ready    <none>          10d

# Confirm control-plane components (they run as static pods, hence namespace kube-system)
kubectl get pods -n kube-system -o wide
# etcd-controlplane                     -> controlplane node
# kube-apiserver-controlplane           -> controlplane node
# kube-scheduler-controlplane           -> controlplane node
# kube-controller-manager-controlplane  -> controlplane node
# kube-proxy-abcde                      -> controlplane node (DaemonSet, runs everywhere)
# kube-proxy-fghij                      -> node01

# Worker-node-only agent, NOT visible via kubectl (it's what runs kubectl's targets)
ssh node01
ps -ef | grep kubelet
# /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml ...

# Container runtime underneath kubelet on that same node
crictl ps
```

**Why this matters:** if `kubectl get nodes` shows `NotReady`, the fault is almost
always **kubelet** or the **container runtime** on that node — not the control plane.
If pods across the *whole cluster* are stuck `Pending`, suspect **kube-scheduler** or
**kube-apiserver**/**etcd** instead.

> ⚠️ **Exam Tip:** don't mix up which of kubelet/kube-proxy is a Pod — **kube-proxy IS
> a Pod** (DaemonSet), **kubelet is NEVER a Pod** (plain OS service). Getting this
> backwards is the single most common architecture-topic trap.

### Key Takeaways / Traps

1. Control-plane components normally run as **static pods** in `kube-system` on
   kubeadm-built clusters — check `/etc/kubernetes/manifests/` on the control-plane
   node to fix them directly.
2. **kubelet is never a pod** — it's a systemd service that must be inspected via
   `service kubelet status` / `journalctl -u kubelet` on the node itself, not
   `kubectl`.
3. **kube-proxy IS a pod** (a DaemonSet), unlike kubelet — easy to mix up on exams.
4. Every arrow in the diagram above either originates or terminates at
   **kube-apiserver** — no component talks to etcd, the scheduler, or another node
   directly.

---

<a id="t03"></a>
## 03. Docker vs ContainerD

> 🎯 **In one sentence:** Kubernetes never runs containers itself — it always talks to
> a pluggable **container runtime** through a standard interface (CRI), and Docker is
> no longer in that path.

**Concept (in plain terms):** Kubernetes delegates the actual "run this container"
work to a **container runtime**, accessed through a standard interface called the
**CRI** (Container Runtime Interface). Docker was the original default; it has since
been replaced by lighter-weight runtimes like **containerd** and **CRI-O**.

### Why Docker was dropped (dockershim)

- Docker itself was never CRI-compliant (it predates CRI and does far more than just
  run containers — networking, builds, CLI, etc.).
- Kubernetes maintained a shim (**dockershim**) translating CRI calls into Docker API
  calls so kubelet could keep using Docker.
- Dockershim was **deprecated in v1.20** and **removed in v1.24**.
- Under the hood, Docker itself was always built on **containerd** — so removing
  Docker support just means kubelet talks to containerd **directly**, skipping the
  unnecessary middle layer.

### Diagram — Then vs. now

```
 OLD (pre-1.24, via dockershim):

   kubelet ──CRI──► dockershim ──Docker API──► dockerd ──► containerd ──► runc ──► container
                     (extra translation layer, removed in 1.24)

 NOW (containerd runtime, direct):

   kubelet ──CRI──► containerd ──► runc (OCI runtime) ──► container

 NOW (CRI-O runtime, direct, alternative to containerd):

   kubelet ──CRI──► CRI-O ──► runc (OCI runtime) ──► container

   Either way: kubelet only ever needs to speak CRI. What implements CRI underneath
   (containerd, CRI-O, or anything else) is a pluggable, swappable detail.
```

### Standards involved

| Standard | Defines | Example implementations |
|---|---|---|
| **OCI (Open Container Initiative)** — Image spec | how a container image is packaged | any OCI-compliant image (Docker images ARE OCI images) |
| **OCI — Runtime spec** | how to actually run a container from an unpacked bundle | `runc` (most common), `crun`, `gVisor`/`runsc` |
| **CRI (Container Runtime Interface)** | gRPC API kubelet uses to talk to *any* runtime | containerd, CRI-O |

### Command-line tools mapping

| Task | Docker CLI | containerd-native | Kubernetes-native (works on any CRI runtime) |
|---|---|---|---|
| List containers | `docker ps` | `ctr containers list` | `crictl ps` |
| List images | `docker images` | `ctr images list` | `crictl images` |
| Pull image | `docker pull nginx` | `ctr images pull docker.io/library/nginx:latest` | `crictl pull nginx` |
| Inspect | `docker inspect <id>` | `ctr containers info <id>` | `crictl inspect <id>` |
| Logs | `docker logs <id>` | — | `crictl logs <id>` |
| Docker-CLI-like UX on raw containerd | `nerdctl ps` (drop-in Docker CLI replacement, talks straight to containerd) | | |

### Worked Example — Diagnosing "which runtime is this node using?"

```bash
kubectl get nodes -o wide
# NAME     ...   CONTAINER-RUNTIME
# node01   ...   containerd://1.6.8

ssh node01
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
# CONTAINER   IMAGE   ...
# (works because containerd speaks CRI natively — no dockershim needed)

# If Docker is still present alongside (common on older nodes), you'd instead see:
docker ps
# but note: `docker ps` output and `crictl ps` output can differ, since kubelet only
# ever talks to the CRI runtime, never to the Docker CLI/daemon layer directly.
```

### Worked Example — Manually pulling and running via crictl (no kubectl involved)

```bash
crictl pull busybox
crictl images
# IMAGE       TAG      IMAGE ID
# busybox     latest   abcd1234

# crictl can create pods/containers directly too, useful for low-level debugging:
crictl runp pod-config.json
crictl create <pod-id> container-config.json pod-config.json
crictl start <container-id>
crictl ps
```

> ⚠️ **Exam Tip:** on any modern node, reach for **`crictl`**, not `docker` — `docker`
> may not even be installed. `crictl`'s subcommands (`ps`, `images`, `logs`,
> `inspect`) deliberately mirror Docker's, so the muscle memory transfers directly.

### Key Takeaways / Traps

1. Kubernetes **never** ran containers directly — even in the Docker era, it always
   went through a runtime; Docker's removal only removed the extra dockershim
   translation hop, since Docker itself sat on containerd all along.
2. `docker run ...` container images still work fine on containerd/CRI-O clusters —
   OCI image format is the same either way. What changed is only the *runtime
   plumbing*, not image compatibility.
3. Use **`crictl`**, not `docker`, to debug containers on a modern (containerd/CRI-O)
   node — `docker` may not even be installed anymore.
4. `nerdctl` exists specifically to give people the familiar Docker CLI experience
   directly on top of containerd, for local/manual use — it's not what kubelet uses.

---

<a id="t04"></a>
## 04. ETCD for Beginners

> 🎯 **In one sentence:** etcd is just a generic, distributed, highly-available
> key-value store — Kubernetes is only one of its users, not something baked into it.

**Concept (in plain terms):** etcd is an open-source, **distributed, consistent
key-value store** used to hold data that needs to be reliably shared across a
distributed system. Kubernetes uses it as the single source of truth for **all**
cluster state.

**Key properties:**

| Property | Detail |
|---|---|
| Data model | **Key-value**, not relational/tabular — just `key -> value` pairs |
| Availability | Runs as a **cluster** of nodes using the **Raft consensus algorithm** to agree on state changes even if some members fail |
| Implementation | Written in Go; ships as a single static binary `etcd` (server) + `etcdctl` (CLI client) |
| Ports | Client requests: **2379**. Peer (inter-etcd) traffic: **2380** |

### Diagram — etcd as a simple key-value store

```
   etcdctl put key1 value1  ─────►  ┌─────────────────────────┐
   etcdctl put key2 value2  ─────►  │           etcd           │
   etcdctl get key1         ◄─────  │  key1 -> value1           │
                                     │  key2 -> value2           │
                                     └─────────────────────────┘

   Raft consensus (HA etcd cluster, 3/5 members):

     etcd-1 ◄──► etcd-2 ◄──► etcd-3        one is elected LEADER;
        ▲            ▲           ▲          writes go through the leader and are
        └────────────┴───────────┘          replicated to followers before being
                                              acknowledged (majority quorum required)
```

### Installing & using etcd standalone (generic, non-Kubernetes)

```bash
# Download & extract (example version)
wget https://github.com/etcd-io/etcd/releases/download/v3.5.9/etcd-v3.5.9-linux-amd64.tar.gz
tar xvf etcd-v3.5.9-linux-amd64.tar.gz
cd etcd-v3.5.9-linux-amd64

# Start the server (foreground, defaults to localhost:2379)
./etcd
```

```bash
# In another terminal — basic operations, using the v3 API
export ETCDCTL_API=3

etcdctl put key1 value1
etcdctl get key1
# key1
# value1

etcdctl put key2 value2
etcdctl get key2

etcdctl del key1

etcdctl get --prefix key   # list all keys starting with "key"
```

### Worked Example — v2 vs v3 API gotcha

```bash
# Without ETCDCTL_API set, older etcdctl builds default to the v2 API, which uses a
# different (and now-deprecated) command set/wire format:
etcdctl set key1 value1        # v2-style command
etcdctl get key1                # ambiguous without knowing which API is active

# ALWAYS set this explicitly before running etcdctl commands during troubleshooting:
export ETCDCTL_API=3
etcdctl version
# etcdctl version: 3.5.9
# API version: 3.5
```

> ⚠️ **Exam Tip:** `export ETCDCTL_API=3` before *any* `etcdctl` command, every single
> time — this one habit prevents the most common etcd mistake: accidentally running
> v2-style syntax against a v3 cluster (or vice versa) and getting confusing errors.

### Key Takeaways / Traps

1. etcd is a **generic** distributed key-value store — it has nothing Kubernetes-
   specific about it; Kubernetes is just one of its consumers (also used by CoreDNS,
   other distributed systems).
2. **Raft** guarantees a *majority* of members must agree before a write is
   committed — this is why etcd clusters are sized odd (1, 3, 5) to always have a
   clear majority possible.
3. Always `export ETCDCTL_API=3` first — mixing v2/v3 commands is one of the most
   common etcd exam/practice traps.
4. Default ports to remember: **2379** (client requests), **2380** (peer/cluster
   communication).

---

<a id="t05"></a>
## 05. ETCD in Kubernetes

> 🎯 **In one sentence:** every object in your cluster is a key under `/registry/...`
> in etcd, and `kube-apiserver` is the only thing ever allowed to read or write it.

**Concept (in plain terms):** Kubernetes stores **every** piece of cluster state in
etcd — nodes, pods, configs, secrets, service accounts, roles/rolebindings, and more —
all under a structured key namespace. **Only `kube-apiserver` ever reads/writes etcd
directly.**

### Diagram — What's actually stored, and who touches it

```
                         kube-apiserver
                               │
                only component that talks to etcd
                               │
                               ▼
                     ┌───────────────────┐
                     │        etcd         │
                     │                     │
                     │ /registry/pods/...  │
                     │ /registry/nodes/... │
                     │ /registry/configmaps/...
                     │ /registry/secrets/...
                     │ /registry/deployments/...
                     │ /registry/replicasets/...
                     │ /registry/serviceaccounts/...
                     │ /registry/roles/...
                     └───────────────────┘

   kube-scheduler, kube-controller-manager, kubelet, kubectl — ALL of these read/write
   cluster state by calling kube-apiserver's REST API, NEVER etcd directly.
```

### HA topology: Stacked vs. External etcd

| Topology | Description | Trade-off |
|---|---|---|
| **Stacked** | etcd runs colocated on the same node(s) as the control plane (`kubeadm` default) | Simpler to set up/manage, but a control-plane node failure risks losing an etcd member too. |
| **External** | etcd runs on its own dedicated cluster of nodes, separate from control-plane nodes | More resilient (etcd failure isolated from apiserver failure), but more infrastructure to manage. |

```
 STACKED:                              EXTERNAL:

 ┌───────────────────┐                 ┌───────────────────┐    ┌──────────────┐
 │ controlplane-node1 │                 │ controlplane-node1 │    │  etcd-node1  │
 │  apiserver + etcd  │                 │  apiserver only    │    │              │
 ├───────────────────┤                 ├───────────────────┤    ├──────────────┤
 │ controlplane-node2 │                 │ controlplane-node2 │    │  etcd-node2  │
 │  apiserver + etcd  │                 │  apiserver only    │    │              │
 ├───────────────────┤                 ├───────────────────┤    ├──────────────┤
 │ controlplane-node3 │                 │ controlplane-node3 │    │  etcd-node3  │
 │  apiserver + etcd  │                 │  apiserver only    │    │              │
 └───────────────────┘                 └───────────────────┘    └──────────────┘
```

### Worked Example — Inspecting etcd directly on a kubeadm cluster

```bash
# etcd itself runs as a static pod — find its manifest for the exact flags/certs used
cat /etc/kubernetes/manifests/etcd.yaml | grep -- --

# Typical relevant flags:
# --advertise-client-urls=https://127.0.0.1:2379
# --cert-file=/etc/kubernetes/pki/etcd/server.crt
# --key-file=/etc/kubernetes/pki/etcd/server.key
# --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
# --data-dir=/var/lib/etcd

# Query etcd directly (from the control-plane node), supplying certs explicitly:
ETCDCTL_API=3 etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -20
# /registry/apiextensions.k8s.io/customresourcedefinitions/...
# /registry/clusterrolebindings/...
# /registry/configmaps/kube-system/coredns
# /registry/namespaces/default
# /registry/pods/default/nginx
# /registry/services/specs/default/kubernetes
# ...

# Confirm this mapping by creating a pod and finding its new key:
kubectl run testpod --image=nginx
ETCDCTL_API=3 etcdctl --endpoints https://127.0.0.1:2379 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/default/testpod --prefix --keys-only
# /registry/pods/default/testpod
```

### Worked Example — Backup and restore (a very common exam task)

```bash
# 1. Take a snapshot backup
ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot-pre-boot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 2. Verify the snapshot
ETCDCTL_API=3 etcdctl snapshot status /opt/snapshot-pre-boot.db -w table

# 3. Restore into a NEW data directory (never overwrite the live one directly)
ETCDCTL_API=3 etcdctl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir /var/lib/etcd-from-backup

# 4. Point etcd's static pod manifest at the new data dir, then restart etcd
#    (edit /etc/kubernetes/manifests/etcd.yaml -> volumes -> hostPath -> path:
#     change from /var/lib/etcd to /var/lib/etcd-from-backup)
#    kubelet detects the manifest change and recreates the etcd static pod automatically.
crictl ps | grep etcd   # confirm it restarted
kubectl get pods --all-namespaces   # confirm cluster state matches the backup point
```

### Worked Example — Checking member health (HA cluster)

```bash
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key -w table
# +------------------+---------+------------------+---------------------------+
# |        ID        | STATUS  |       NAME       |        PEER ADDRS         |
# +------------------+---------+------------------+---------------------------+
# | 8211f1d0f64f3269 | started | controlplane      | https://127.0.0.1:2380    |
# +------------------+---------+------------------+---------------------------+
```

> ⚠️ **Exam Tip:** restoring a snapshot **always** goes into a brand-new
> `--data-dir`, then you repoint etcd's static pod manifest at it — never restore
> in-place onto the live data directory, and never skip finding the real cert paths
> from `/etc/kubernetes/manifests/etcd.yaml` first.

### Key Takeaways / Traps

1. **kube-apiserver is the only client of etcd** in a Kubernetes cluster — never query
   or modify etcd directly except for backup/restore/debugging.
2. Restoring a snapshot always goes to a **new** `--data-dir`; then you repoint the
   static pod manifest — never restore in-place onto the live etcd data directory.
3. Certificates are mandatory for every `etcdctl` call against a real cluster
   (`--cacert`, `--cert`, `--key`) — find the exact paths from
   `/etc/kubernetes/manifests/etcd.yaml`, don't guess/hardcode them.
4. **Stacked etcd** (etcd colocated with control plane) is the `kubeadm` default;
   **external etcd** trades setup simplicity for better failure isolation.
5. `etcdctl snapshot save` requires talking to a **running** etcd endpoint; if etcd
   itself is down, you must instead copy the raw `--data-dir` files directly from disk.
6. All Kubernetes object keys live under the `/registry/` prefix in etcd — useful for
   sanity-checking that an object you created via `kubectl` actually persisted.

---

<a id="t06"></a>
## 06. Kube-API Server

> 🎯 **In one sentence:** every single request — from `kubectl`, controllers, or the
> outside world — passes through `kube-apiserver`'s Authenticate → Authorize →
> Admission → etcd-write pipeline before anything happens.

**Concept (in plain terms):** The API server is the **only** entry point into the
cluster. Every `kubectl` command, every controller, the scheduler, kubelet,
kube-proxy, and any external client (dashboard, CI/CD, custom controllers) all talk
to it over its REST API — it is the sole component that reads/writes etcd.

**What it does with every request, in order:**

| Step | Question it answers |
|---|---|
| 1. **Authentication** | Who are you? (certs, tokens, etc.) |
| 2. **Authorization** | Are you allowed to do this? (RBAC, ABAC, etc.) |
| 3. **Admission Control** | Should this request be mutated or rejected? (e.g. `LimitRanger`, `ResourceQuota`, `NamespaceLifecycle` admission controllers) |
| 4. **Persist to etcd** | Write the validated object. |
| 5. **Notify watchers** | Scheduler/controller-manager/kubelet get notified via the watch mechanism so they can react. |

### Diagram — Anatomy of a single `kubectl create` call

```
 kubectl create -f pod.yaml
         │
         ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                          kube-apiserver                              │
 │                                                                       │
 │   1. Authenticate  ──►  2. Authorize (RBAC)  ──►  3. Admission        │
 │      (cert/token)          (can this user            Controllers     │
 │                             create pods here?)       (mutate/reject) │
 │                                                          │            │
 │                                                          ▼            │
 │                                                4. Write object to etcd│
 └─────────────────────────────────────────────────────┬─────────────────┘
                                                         │ watch notification
                                     ┌───────────────────┼───────────────────┐
                                     ▼                                       ▼
                            kube-scheduler                         kube-controller-manager
                        (sees new unscheduled Pod,               (would react here for
                         picks a node, writes Binding             ReplicaSet/Deployment-
                         back through apiserver)                  owned objects)
```

### Worked Example — Finding and reading the live apiserver config

```bash
# On a kubeadm cluster, apiserver runs as a static pod:
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -- "- --"
# --advertise-address=10.0.0.5
# --authorization-mode=Node,RBAC
# --client-ca-file=/etc/kubernetes/pki/ca.crt
# --etcd-servers=https://127.0.0.1:2379
# --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
# --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
# --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
# --service-cluster-ip-range=10.96.0.0/12
# --secure-port=6443

kubectl get pods -n kube-system | grep apiserver
# kube-apiserver-controlplane   Running

# Confirm it's the single front door: every other control-plane pod's logs show it
# calling back to apiserver, never to etcd/each other directly
kubectl logs kube-scheduler-controlplane -n kube-system | grep -i "https://"
```

### Worked Example — Tracing auth+admission failure

```bash
# A user tries to create a pod but lacks RBAC permission:
kubectl auth can-i create pods --as=jane --namespace=dev
# no

kubectl create -f pod.yaml --as=jane -n dev
# Error from server (Forbidden): pods is forbidden: User "jane" cannot create
# resource "pods" in API group "" in the namespace "dev"

# This request never even reached admission control or etcd — it was rejected at the
# Authorization stage, exactly as the flow diagram shows.
```

> ⚠️ **Exam Tip:** if `kubectl` hangs or times out **cluster-wide** (not just on one
> node), suspect `kube-apiserver` or `etcd` first — literally nothing else can
> function without them, so don't waste time checking individual pods/nodes yet.

### Key Takeaways / Traps

1. `--authorization-mode` order matters — `Node,RBAC` means Node authorizer runs
   first, then RBAC; a request must pass **every** configured mode's applicable checks.
2. The apiserver's own client cert to talk to **etcd** (`--etcd-certfile`/
   `--etcd-keyfile`) is a *different* cert pair than the one clients use to talk to
   *it* (`--client-ca-file`, `--tls-cert-file`) — don't confuse the two when
   troubleshooting cert errors.
3. If `kubectl` hangs or times out cluster-wide (not just one node), suspect
   **kube-apiserver** or **etcd**, since literally nothing else can function without
   it.
4. Admission controllers can **mutate** objects (e.g. auto-inject defaults/sidecars)
   as well as just validate — always check `kubectl get <obj> -o yaml` for
   fields you didn't write yourself (e.g. auto-filled resource requests from a
   `LimitRange`).

---

<a id="t07"></a>
## 07. Kube-Controller-Manager

> 🎯 **In one sentence:** one process, many independent "control loops" bundled
> together — each one watches the apiserver for a specific kind of object and nudges
> reality back toward the desired state.

**Concept (in plain terms):** A single binary/process that bundles together **many
independent control loops ("controllers")**, each responsible for watching the
apiserver for one kind of object and reconciling actual state toward desired state.

**Common built-in controllers:**

| Controller | Watches / Reconciles |
|---|---|
| **Node Controller** | Node health; marks `NotReady`, evicts pods after a grace period if a node stops heartbeating |
| **Replication Controller** | Ensures the right number of pods exist for ReplicationControllers |
| **ReplicaSet / Deployment controllers** | Same idea for ReplicaSets/Deployments (rollout logic) |
| **Job Controller** | Ensures Job pods run to completion |
| **Endpoints Controller** | Populates Endpoints objects (joins Services ↔ Pods) |
| **Namespace Controller** | Cleans up all objects when a Namespace is deleted |
| **Service Account & Token Controllers** | Create default ServiceAccounts and their tokens for new Namespaces |

### Diagram — Node failure, handled by Node Controller

```
   node02 stops sending heartbeats to kube-apiserver
                        │
                        ▼
   Node Controller (inside kube-controller-manager) notices, after
   --node-monitor-grace-period (default 40s):
                        │
                        ▼
   marks node02 status "Ready" -> "Unknown", adds taint:
        node.kubernetes.io/unreachable:NoExecute
                        │
                        ▼
   after --pod-eviction-timeout (default 5m) with no recovery:
   pods on node02 (not tolerating that taint) are evicted and
   rescheduled elsewhere by kube-scheduler
```

### Worked Example — Inspecting flags and watching a controller react

```bash
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep -- "- --"
# --node-monitor-period=5s
# --node-monitor-grace-period=40s
# --pod-eviction-timeout=5m0s
# --controllers=*                     # all controllers enabled (default)
# --leader-elect=true                 # HA: only one controller-manager is active at a time

# Simulate a node failure and watch the Node Controller react:
ssh node02
service kubelet stop

# Back on control plane:
kubectl get nodes -w
# node02   Ready      -> after ~40s ->   NotReady

kubectl describe node node02 | grep Taints
# Taints: node.kubernetes.io/unreachable:NoExecute

kubectl get pods -o wide
# pods previously on node02 eventually show as evicted/rescheduled onto healthy nodes
```

### Worked Example — Selectively disabling a controller

```yaml
# in kube-controller-manager.yaml, under command args:
- --controllers=*,-nodeipam    # runs everything EXCEPT the nodeipam controller
```
```bash
# after editing the static pod manifest, kubelet auto-restarts the pod:
kubectl get pods -n kube-system | grep controller-manager
```

> ⚠️ **Exam Tip:** don't conflate the two node-failure timers — **grace period**
> (default 40s, how long before marking `NotReady`) and **eviction timeout** (default
> 5m, how long after that before evicting pods) are two separate flags with two
> separate jobs.

### Key Takeaways / Traps

1. **One process, many controllers** — don't confuse "controller-manager" (the
   process/pod) with an individual "controller" (one control loop inside it).
2. `--leader-elect=true` matters in HA setups: multiple `kube-controller-manager`
   replicas run, but only the elected **leader** is actively reconciling at any time —
   the rest are standby.
3. Node eviction timing is controlled by **two separate flags**:
   `--node-monitor-grace-period` (how long before marking `NotReady`) and
   `--pod-eviction-timeout` (how long after that before evicting pods) — don't
   conflate them.
4. Every controller here only ever talks to **kube-apiserver**, same golden rule as
   everything else in the cluster.

---
