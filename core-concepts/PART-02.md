
<a id="t08"></a>
## 08. Kube-Scheduler

> 🎯 **In one sentence:** for every Pod with no node assigned, the scheduler runs a
> Filter → Score pipeline to pick the best available node.

**Concept (in plain terms):** Watches for Pods with `spec.nodeName` unset and assigns
them to the best available node via a two-phase **Filter → Score** pipeline.

> **Full deep dive already covered separately** — see
> `kubernetes-scheduling-notes.md` for: manual scheduling, labels/selectors, taints &
> tolerations, node selectors/affinity, resource-based filtering, DaemonSets, static
> pods, multiple schedulers, and scheduler profiles/configuration, each with worked
> examples. This section keeps only the core-concepts-level summary.

### Diagram — Quick recap

```
   Pod created, spec.nodeName empty
                │
                ▼
   ┌─────────────────────┐      ┌─────────────────────┐
   │      FILTERING       │ ──► │       SCORING         │ ──► winning node ──► Binding
   │  (drop infeasible     │     │  (rank remaining       │      written via apiserver
   │   nodes: resources,   │     │   feasible nodes)      │
   │   taints, affinity)   │     └─────────────────────┘
   └─────────────────────┘
```

### Worked Example — Confirming it's a static pod, like the others

```bash
cat /etc/kubernetes/manifests/kube-scheduler.yaml | grep -- "- --"
# --leader-elect=true
# --kubeconfig=/etc/kubernetes/scheduler.conf
# --bind-address=127.0.0.1

kubectl get pods -n kube-system | grep scheduler
# kube-scheduler-controlplane   Running

kubectl get events --field-selector reason=Scheduled
# LAST SEEN   TYPE     REASON      OBJECT       MESSAGE
# 5s          Normal   Scheduled   pod/nginx    Successfully assigned default/nginx to node01
```

> ⚠️ **Exam Tip:** a Pod stuck `Pending` with **no** `Scheduled` event at all means
> the scheduler itself isn't running (or a custom `schedulerName` has no matching
> scheduler); a Pod stuck `Pending` **with** filter-failure events means the scheduler
> ran fine but found no feasible node — two very different root causes, same symptom.

### Key Takeaways / Traps

1. Like apiserver/controller-manager/etcd, kube-scheduler runs as a **static pod**
   on kubeadm control-plane nodes — check `/etc/kubernetes/manifests/` first for
   config issues.
2. `--leader-elect=true` applies here too for HA control planes — only one scheduler
   replica is actively binding pods at a time.
3. A Pod stuck `Pending` with **no** `Scheduled` event at all usually means the
   scheduler itself isn't running (or a custom `schedulerName` has no matching
   scheduler) — a Pod stuck `Pending` **with** filter-failure events in
   `kubectl describe pod` means the scheduler ran but found no feasible node.

---

<a id="t09"></a>
## 09. Kubelet

> 🎯 **In one sentence:** kubelet is the one control-plane-adjacent piece that is
> **not** a pod — it has to be alive first, since it's the thing that creates all the
> other static pods.

**Concept (in plain terms):** The **primary node agent**. It's the only
control-plane-adjacent component that is **not** managed as a static pod by
Kubernetes itself — kubelet has to already be running (as a plain OS service)
*before* it can even create the other static pods, since it's the one reading the
static manifest directory in the first place.

**What kubelet actually does:**

| Step | Action |
|---|---|
| 1 | Registers its node with the apiserver (creates/updates the Node object) |
| 2 | Watches the apiserver for Pods assigned to (`spec.nodeName ==`) its own node |
| 3 | Calls the **container runtime** (via CRI) to pull images and start/stop containers |
| 4 | Calls **CNI** plugins for pod networking, **CSI** for volumes |
| 5 | Continuously reports node & pod status back to the apiserver (heartbeats — what Node Controller in topic 07 is watching) |
| 6 | Runs periodic liveness/readiness/startup **probes** against containers |

### Diagram — Kubelet's position: the one non-pod component

```
   ┌───────────────────────────────────────────────────────────┐
   │                     Worker Node                            │
   │                                                              │
   │   kubelet   <── plain systemd/OS service, installed          │
   │      │           manually (or via kubeadm join), NOT a pod   │
   │      │                                                        │
   │      ├──watches Pods for this node──► kube-apiserver          │
   │      │                                                        │
   │      ├──CRI──► container runtime (containerd/CRI-O)           │
   │      │                                                        │
   │      ├──CNI──► pod networking setup                           │
   │      │                                                        │
   │      ├──CSI──► volume mount/unmount                           │
   │      │                                                        │
   │      └──reads──► /etc/kubernetes/manifests/*.yaml             │
   │                    (static pods: THIS is how apiserver,       │
   │                     etcd, scheduler, controller-manager       │
   │                     get started on control-plane nodes —      │
   │                     kubelet creates them, not the reverse!)   │
   └───────────────────────────────────────────────────────────┘
```

### Worked Example — Installing kubelet manually ("the hard way" style)

```bash
# Download the binary directly (illustrative version)
wget https://storage.googleapis.com/kubernetes-release/release/v1.28.0/bin/linux/amd64/kubelet
chmod +x kubelet
mv kubelet /usr/local/bin/

# kubelet needs a config file and a systemd unit — key config fields:
cat /var/lib/kubelet/config.yaml
# apiVersion: kubelet.config.k8s.io/v1beta1
# kind: KubeletConfiguration
# authentication:
#   x509:
#     clientCAFile: /etc/kubernetes/pki/ca.crt
# staticPodPath: /etc/kubernetes/manifests
# clusterDNS: ["10.96.0.10"]

cat /etc/systemd/system/kubelet.service
# ExecStart=/usr/local/bin/kubelet \
#   --config=/var/lib/kubelet/config.yaml \
#   --kubeconfig=/etc/kubernetes/kubelet.conf \
#   --container-runtime-endpoint=unix:///run/containerd/containerd.sock

systemctl daemon-reload
systemctl enable --now kubelet
systemctl status kubelet
```

### Worked Example — Diagnosing a NotReady node (kubelet-side)

```bash
kubectl get nodes
# node01   NotReady

ssh node01
systemctl status kubelet
# inactive (dead)   <- found it

journalctl -u kubelet -n 50 --no-pager
# look for cert errors, wrong --kubeconfig path, container runtime socket not found, etc.

systemctl restart kubelet
kubectl get nodes
# node01   Ready
```

> ⚠️ **Exam Tip:** node `NotReady`? Go straight to **kubelet on that node**
> (`systemctl status kubelet`, `journalctl -u kubelet`) — a dead kubelet can't report
> anything back to `kubectl`, so cluster-side commands won't tell you why.

### Key Takeaways / Traps

1. **Kubelet is the only piece here that is never a static/regular pod** — it must
   already be alive to create the pods that host apiserver/etcd/scheduler/
   controller-manager. This is the single most common "which component is different"
   exam trap in this whole section.
2. `staticPodPath` in kubelet's config is exactly the directory
   (`/etc/kubernetes/manifests` by default) that ties back to the Static Pods topic —
   kubelet is the thing actually watching that folder.
3. Node `NotReady` → always check **kubelet's own service status/logs on that node
   first** (`systemctl status kubelet`, `journalctl -u kubelet`), not `kubectl`
   commands, since a dead kubelet can't even report why.
4. `--container-runtime-endpoint` must point at the correct CRI socket
   (`containerd.sock`, `crio.sock`, etc.) — mismatches here are a classic
   post-migration (Docker→containerd) breakage.

---

<a id="t10"></a>
## 10. Kube-Proxy

> 🎯 **In one sentence:** kube-proxy is a DaemonSet pod on every node that writes the
> networking rules making a Service's virtual IP actually route to real Pods.

**Concept (in plain terms):** A **DaemonSet** (one pod per node) that maintains
network rules on each node so traffic sent to a **Service** gets routed to one of its
backing **Pods** — implementing the "virtual IP" behavior of Services.

**Modes (implementation of the actual rule-writing):**

| Mode | How it works | Notes |
|---|---|---|
| `iptables` (default) | Writes `iptables` NAT rules per Service/Endpoint | Simple, but rule count scales linearly with Services — slower at very large scale |
| `IPVS` | Uses Linux IPVS (kernel load-balancer) | Better performance/scaling for large clusters, supports more LB algorithms |
| `userspace` (legacy) | Proxies connections in userspace | Deprecated, far slower — essentially unused today |

### Diagram — How kube-proxy makes a Service's ClusterIP actually work

```
   Client Pod ──► curl http://<service-clusterIP>:80
                          │
                          ▼
        node's iptables/IPVS rules (written by kube-proxy)
        intercept traffic to that ClusterIP:port and DNAT it to
        one of the Service's actual backend Pod IPs (round-robin/random)
                          │
                          ▼
                 Pod backend-1 (10.244.1.5:8080)   ◄── chosen this time
                 Pod backend-2 (10.244.2.7:8080)
                 Pod backend-3 (10.244.3.9:8080)

   kube-proxy watches apiserver for Service & Endpoints/EndpointSlice changes and
   rewrites these rules continuously — nothing here is static.
```

### Worked Example — Inspecting kube-proxy and its rules

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
# kube-proxy-abcde   node01
# kube-proxy-fghij   node02

kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
#   mode: "iptables"

# On any node, see the actual rules kube-proxy generated for a Service:
iptables-save | grep KUBE-SERVICES | head -5
iptables-save | grep <service-clusterIP>
```

### Worked Example — Switching to IPVS mode

```bash
kubectl edit configmap kube-proxy -n kube-system
# change:
#   mode: ""
# to:
#   mode: "ipvs"

# Restart kube-proxy pods so the change takes effect (DaemonSet, so delete-all is safe,
# they're immediately recreated):
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# Verify:
ssh node01
ipvsadm -Ln
# TCP  10.96.0.1:443 rr
#   -> 10.0.0.5:6443    Masq   1  0   0
```

### Worked Example — Diagnosing "Service not reachable" via kube-proxy

```bash
kubectl get endpoints myservice
# NAME        ENDPOINTS
# myservice   <none>          <- no pods matched the Service's selector! (not a kube-proxy bug)

kubectl get pods -l app=myapp --show-labels
# compare against myservice's spec.selector — classic label mismatch, same class of bug
# as the Deployment selector mismatch trap in the ReplicaSets section.

# If Endpoints DO list pod IPs but traffic still fails, THEN suspect kube-proxy itself:
kubectl logs -n kube-system kube-proxy-abcde
systemctl status kube-proxy    # N/A — it's a pod, not a systemd service (unlike kubelet!)
```

> ⚠️ **Exam Tip:** empty `Endpoints` on a Service is a **selector/label mismatch**,
> not a kube-proxy problem — always check `kubectl get endpoints <svc>` before you
> even think about blaming kube-proxy itself.

### Key Takeaways / Traps

1. Kube-proxy **is a Pod** (DaemonSet-managed) — unlike kubelet, which is a plain OS
   service. Easy to flip these two in your head; don't.
2. It only ever reacts to **Service** and **Endpoints/EndpointSlice** changes — if a
   Service has empty Endpoints, the problem is the Service's `selector` not matching
   any Pod labels, **not** kube-proxy.
3. `iptables` mode is still the default in most distros; `IPVS` needs the kernel IPVS
   modules loaded on every node (`ipvsadm`, `ip_vs*` kernel modules) or the mode
   silently falls back.
4. Changing kube-proxy's mode requires **restarting its pods** (editing the
   ConfigMap alone doesn't hot-reload) — delete-and-let-DaemonSet-recreate is the
   simplest way.

---

<a id="t11"></a>
## 11. Pods

> 🎯 **In one sentence:** a Pod is the smallest deployable unit — Kubernetes never
> runs a bare container, it always wraps one (or a few tightly-coupled ones) in a Pod.

**Concept (in plain terms):** A **Pod** is the smallest deployable unit in
Kubernetes. Kubernetes never schedules a bare container onto a node — every
container, even a lone one, is always wrapped inside a Pod first. Most of the time
this is a **one-to-one** relationship (one Pod, one container), but a Pod can hold
**multiple containers** when those containers are tightly coupled and need to share
the same network namespace and storage volumes (e.g. a main app container plus a
logging/helper sidecar) — not usually multiple containers of the *same* kind.

**Pod definition YAML anatomy — four required top-level fields, always:**

| Field | Purpose |
|---|---|
| `apiVersion` | Which API version this object schema belongs to (`v1` for Pods) |
| `kind` | What kind of object this is (`Pod`) — **case-sensitive** |
| `metadata` | Data *about* the object: `name`, `labels` (key-value pairs used later for grouping/selection by ReplicaSets, Services, etc.) |
| `spec` | The actual desired state: for a Pod, a `containers` list (each with `name`, `image`, optionally `ports`, etc.) |

### Diagram

```
 Kubernetes NEVER schedules a bare container — every container is always wrapped in
 at least one Pod, even for a single-container app:

   ┌─────────────────────────────┐        ┌───────────────────────────────────────┐
   │   Single-container Pod       │        │        Multi-container Pod             │
   │  ┌─────────────────────┐    │        │  ┌───────────┐    ┌────────────────┐  │
   │  │   nginx container    │    │        │  │   nginx    │    │    agentx       │  │
   │  └─────────────────────┘    │        │  │ (main app) │    │ (helper/logger) │  │
   │        1 Pod == 1 IP,        │        │  └───────────┘    └────────────────┘  │
   │        1 network namespace   │        │      share: network namespace,          │
   └─────────────────────────────┘        │      localhost, storage volumes         │
                                            └───────────────────────────────────────┘

   Worker Node
   ┌───────────────────────────────────────────────────────────────────────┐
   │   Pod "nginx" (1 container)    Pod "webapp" (2 containers)             │
   └───────────────────────────────────────────────────────────────────────┘
```

### Worked Example — Deploying and inspecting a Pod imperatively

```bash
# The fastest way to get a single Pod running — no YAML needed
kubectl run nginx --image=nginx

kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# nginx   1/1     Running   0          5s

kubectl get pods -o wide
# NAME    READY   STATUS    IP           NODE
# nginx   1/1     Running   10.244.0.5   node01

kubectl describe pod nginx
# Name, Node, Labels, IP, Containers (Image, State, Ready), Events
```

### Worked Example — Pod definition YAML, written out and created

```yaml
# pod-definition.yaml
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

```bash
kubectl create -f pod-definition.yaml
kubectl get pods
# myapp-pod   1/1   Running
```

### Worked Example — A broken multi-container Pod (`webapp`), diagnosed like the exam

```bash
kubectl describe pod webapp
# Containers:
#   nginx:
#     Image:  nginx
#     State:  Running
#   agentx:
#     Image:  agentx
#     State:  Waiting
#     Reason: ImagePullBackOff
# Events:
#   Warning  Failed   ...  Failed to pull image "agentx": ...failed to pull and unpack image...

kubectl get pods
# NAME     READY   STATUS             RESTARTS
# webapp   1/2     ImagePullBackOff   0
```

`READY` here is **containers-ready / containers-total for that Pod** — `1/2` means
one of the two containers (`nginx`) is up and the other (`agentx`) is not; it is
**not** a count of how many Pods are ready.

### Worked Example — Generating YAML fast, fixing a bad image three ways

```bash
# Generate a starter YAML instead of hand-writing it, without actually creating anything:
kubectl run redis --image=redis123 --dry-run=client -o yaml > redis.yaml
kubectl create -f redis.yaml
# Pod comes up with ImagePullBackOff since redis123 doesn't exist

# Method 1 — edit the YAML, then apply
vi redis.yaml        # fix image: redis123 -> redis
kubectl apply -f redis.yaml

# Method 2 — edit the live Pod directly (only certain fields are actually editable)
kubectl edit pod redis
# change image, save+exit; if the edit is invalid you're dropped back into vi

# Method 3 — patch just the image, by container name, no YAML at all
kubectl set image pod/redis redis=redis
```

### Worked Example — Logs, exec, and cleanup

```bash
kubectl logs nginx                  # single-container pod
kubectl logs webapp -c agentx       # multi-container pod: must name the container

kubectl exec -it nginx -- /bin/bash

kubectl delete pod webapp
kubectl delete pod webapp --force --grace-period=0   # skip graceful termination/confirmation
```

> ⚠️ **Exam Tip:** `kubectl describe pod <name>` is your single most useful
> troubleshooting command — always read the **Events** section at the bottom first;
> it names the real root cause (e.g. `Failed to pull and unpack image`) far faster
> than guessing from the `STATUS` column alone.

### Key Takeaways / Traps

1. Kubernetes **always** wraps containers in a Pod — there is no such thing as a
   container running directly on a node under Kubernetes' management.
2. `READY` in `kubectl get pods` is **ready-containers/total-containers for that one
   Pod**, not a pod count — a Pod with 2 containers where only 1 is up shows `1/2` and
   is *not* considered Running/healthy overall.
3. `kubectl describe pod <name>` is the single most useful troubleshooting command —
   the **Events** section at the very bottom almost always names the real root cause
   (e.g. `Failed to pull and unpack image`), so check it before guessing.
4. `kubectl describe pod <name> | grep -i image` and `kubectl get pods --no-headers |
   wc -l` are fast filtering tricks worth having muscle memory for on the exam.
5. `kubectl run <name> --image=<image> --dry-run=client -o yaml > file.yaml` is the
   standard way to generate a correct YAML skeleton fast instead of hand-typing it —
   used constantly for both Pods and other objects.
6. Once a Pod is running, **most `spec` fields are immutable** — in practice only
   things like the container `image`, `spec.activeDeadlineSeconds`, and
   `spec.tolerations` can be changed via `kubectl edit pod`. Anything else requires
   exporting the YAML, deleting, and recreating the Pod.
7. `kubectl set image pod/<pod> <container-name>=<new-image>` patches an image without
   touching YAML at all — but you must know the **container's name**, not the pod's.
8. `kubectl delete pod <name> --force --grace-period=0` skips graceful termination —
   fine for burning time in an exam/lab, but avoid on real production Pods since
   cleanup hooks won't run.

---

<a id="t12"></a>
## 12. Practice Test Introduction

> 🎯 **In one sentence:** a pure orientation lecture — no Kubernetes content, just
> "here's how the hands-on lab environment works."

This lecture is purely meta/instructional — it has no Kubernetes technical content of
its own. It just walks through how the hands-on practice-test labs work in the course
(how to launch the interactive lab environment, where to find questions, and how
solutions/hints are structured) so students know what to expect before starting the
actual Pods/ReplicaSets/Deployments practice tests that follow.

---

<a id="t13"></a>
## 13. Practice Test - PODs

> 🎯 **In one sentence:** a hands-on drill of everything in topic 11 — nothing new,
> just practice.

**What this test drills:** a full pass through the Pod lifecycle — counting/inspecting
existing pods, creating one imperatively, diagnosing a broken multi-container pod, and
fixing a bad image three different ways. There is no new concept here beyond section
11 — every lesson has already been folded into that section's **Key Takeaways**; this
is the condensed list of exactly what the test exercises:

| Command | What it drills |
|---|---|
| `kubectl get pods` / `kubectl get pods --no-headers \| wc -l` | Count pods without eyeballing the table |
| `kubectl run nginx --image=nginx` | Create a pod imperatively, no YAML |
| `kubectl describe pod newpods \| grep image` | Confirm which image a set of same-named pods is using, in one line |
| `kubectl get pods -o wide` | See which **node** each pod landed on |
| `kubectl describe pod webapp` | For a multi-container pod: read `Containers:` per-container, then `State:`, then `Events:` at the bottom to find *why* `agentx` is `ImagePullBackOff` |
| Reading `READY` correctly | `1/2` means one of *two containers in that single Pod* is ready — a very common misreading trap |
| `kubectl delete pod webapp` / `--force` | Deleting is instant to fix broken pods in a lab; `--force` skips graceful shutdown (fine in exam/lab, risky in production) |
| `kubectl run redis --image=redis123 --dry-run=client -o yaml > redis.yaml` then `kubectl create -f redis.yaml` | The standard "generate then create" pattern for building YAML fast |
| Fixing a bad image **three equivalent ways** | Edit YAML + `kubectl apply -f`, `kubectl edit pod` directly, or `kubectl set image pod/<pod> <container>=<image>` |

---

<a id="t14"></a>
## 14. ReplicaSets

> 🎯 **In one sentence:** a ReplicaSet's entire job is to keep exactly N identical
> Pods alive at all times — delete one and it's replaced almost instantly.

**Concept (in plain terms):** A **ReplicaSet** is a controller whose entire job is to
ensure a **specified number of identical Pod replicas** are running at all times — if
a Pod dies, disappears, or is deleted, the ReplicaSet's control loop notices the
mismatch between desired and actual count and creates a replacement automatically.
This is Kubernetes' answer to "what happens if my one Pod crashes?"

**ReplicationController vs ReplicaSet:**

| | ReplicationController (older) | ReplicaSet (modern) |
|---|---|---|
| `apiVersion` | `v1` | `apps/v1` |
| `kind` | `ReplicationController` | `ReplicaSet` |
| `spec.selector` | Optional — defaults to the Pod template's own `labels` if omitted | **Required** — must be written explicitly, and must match the template's `labels` |

Use **ReplicaSet** going forward; ReplicationController is legacy.

**Definition file anatomy (both look almost identical):**

```yaml
# rc-definition.yaml (ReplicationController — older)
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
```

```yaml
# replicaset-definition.yaml (ReplicaSet — modern)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
  selector:
    matchLabels:
      type: front-end
```

Note the extra `selector.matchLabels` block — this is the one structural field
ReplicaSet adds over ReplicationController, and it must match the Pod template's own
`labels` (or the ReplicaSet will refuse to consider its own template's pods as its
own).

### Diagram

```
 Desired state (replicas: 3) vs actual state — the controller's whole job:

   replicaset-definition.yaml              ReplicaSet "myapp-replicaset"
   spec.replicas: 3              ──create──►   watches pods matching
   spec.selector.matchLabels:                   selector.matchLabels: {type: front-end}
     type: front-end                                     │
                                                           ▼
                                       ┌────────────────────────────────────┐
                                       │ Pod-1 (type=front-end)   Running    │
                                       │ Pod-2 (type=front-end)   Running    │
                                       │ Pod-3 (type=front-end)   Running    │
                                       └────────────────────────────────────┘

   kubectl delete pod Pod-2
                          │
                          ▼
   ReplicaSet controller notices actual (2) != desired (3)
                          │
                          ▼
   creates a replacement Pod-4 (type=front-end) automatically   <- self-healing

   IMPORTANT: selector.matchLabels also lets a ReplicaSet ADOPT any pre-existing pods
   that merely carry matching labels — it isn't limited to pods it created itself.
```

### Worked Example — Creating an RC and an RS, side by side

```bash
kubectl create -f rc-definition.yaml
kubectl get replicationcontroller
kubectl get pods

kubectl create -f replicaset-definition.yaml
kubectl get replicaset
# NAME               DESIRED   CURRENT   READY   AGE
# myapp-replicaset   3         3         3       10s
kubectl get pods
```

### Worked Example — Self-healing in action

```bash
kubectl get pods
# new-replica-set-abcde   Running
# new-replica-set-fghij   Running
# new-replica-set-klmno   Running
# new-replica-set-pqrst   Running     (4 total, desired: 4)

kubectl delete pod new-replica-set-abcde

kubectl get pods
# still shows exactly 4 pods — a brand-new one replaced the deleted one immediately.
# "Why are there still 4 PODs, even after you deleted one?" -> ReplicaSets ensure the
# desired number of PODs always run.
```

### Worked Example — Scaling three ways

```bash
# 1. Edit replicas: in the definition file, then re-apply
vi replicaset-definition.yaml     # change replicas: 3 -> replicas: 6
kubectl apply -f replicaset-definition.yaml

# 2. kubectl scale against the file
kubectl scale --replicas=6 -f replicaset-definition.yaml

# 3. kubectl scale against the live object by type + name (no file needed)
kubectl scale --replicas=6 replicaset myapp-replicaset
```

### Worked Example — Fixing broken ReplicaSet YAML (exam-style traps)

```bash
# Trap #1: wrong apiVersion
kubectl create -f replicaset-definition-1.yaml
# error: unable to recognize ... no matches for kind "ReplicaSet" in version "extensions/v1beta1"

kubectl explain replicaset | grep VERSION
# VERSION:  apps/v1

vi replicaset-definition-1.yaml     # fix apiVersion: apps/v1
kubectl create -f replicaset-definition-1.yaml

# Trap #2: selector.matchLabels doesn't match the Pod template's labels
kubectl create -f replicaset-definition-2.yaml
# error: `selector` does not match template `labels`

vi replicaset-definition-2.yaml     # make matchLabels: {...} equal template.metadata.labels
kubectl create -f replicaset-definition-2.yaml

# Cleanup — delete multiple ReplicaSets in one command
kubectl delete replicaset replicaset-1 replicaset-2
```

### Worked Example — "ReplicaSets aren't smart": editing the image doesn't fix running Pods

```bash
kubectl edit replicaset new-replica-set
# change spec.template.spec.containers[0].image to the correct value, save+exit

kubectl get pods
# still shows the OLD broken pods — editing a ReplicaSet's template does NOT
# retroactively update pods it already created.

# Fix option A — delete + recreate the ReplicaSet from its own exported YAML
kubectl get rs new-replica-set -o yaml > rs.yaml
kubectl delete rs new-replica-set
kubectl create -f rs.yaml

# Fix option B — delete each broken pod individually; RS deploys a fresh (fixed) one
kubectl delete pod new-replica-set-xxxx

# Fix option C — scale to 0, then back up; forces every pod to be recreated from
# the now-corrected template
kubectl scale rs new-replica-set --replicas 0
kubectl scale rs new-replica-set --replicas 4
```

> ⚠️ **Exam Tip:** editing a running ReplicaSet's `image` **never** touches its
> already-running Pods — ReplicaSets only enforce Pod *count*, not spec consistency.
> To roll out a new image you must force Pod recreation yourself (delete pods,
> scale-to-0-and-back, or recreate the RS) — or just use a **Deployment** instead.

### Key Takeaways / Traps

1. **ReplicaSet requires `spec.selector.matchLabels`**; ReplicationController does
   not (it falls back to the template's own labels) — this is the one structural YAML
   difference between the two.
2. `selector.matchLabels` must exactly match `spec.template.metadata.labels`, or
   creation fails outright with a `selector does not match template labels` error —
   a very common hand-typed-YAML trap.
3. Deleting a Pod that a ReplicaSet manages **never** reduces the visible pod count —
   a replacement appears almost immediately. This *is* the self-healing feature, not
   a bug.
4. **Editing a running ReplicaSet's `image` does not touch already-running Pods** —
   ReplicaSets only enforce *count*, not *pod spec consistency*. To roll out a new
   image you must delete the old pods (individually, via scale-to-0-and-back, or by
   deleting and recreating the whole ReplicaSet) — or better, use a **Deployment**
   instead, which handles exactly this problem (see next section).
5. `kubectl explain <kind> | grep VERSION` is the fastest way to find the correct
   `apiVersion` for any resource when `kubectl create` fails with "no matches for
   kind".
6. `kubectl get rs -o wide` shows the `IMAGES` column directly; `kubectl describe
   replicaset` shows it under `Containers:` — either works, `rs` is valid shorthand
   for `replicaset` everywhere.
7. `kubectl delete replicaset name1 name2` (or `kubectl delete rs name1 name2`)
   deletes multiple ReplicaSets in a single command.

---

<a id="t15"></a>
## 15. Practice Tests - ReplicaSet

> 🎯 **In one sentence:** a hands-on drill of everything in topic 14 — nothing new,
> just practice.

**What this test drills:** counting existing ReplicaSets/Pods, reading `DESIRED` vs
`READY` columns, diagnosing why pods aren't ready, confirming self-healing by deleting
a pod, fixing two intentionally-broken ReplicaSet definition files (bad `apiVersion`,
then mismatched `selector`/template labels), and the "editing the image doesn't fix
running pods" gotcha. Every lesson here is already folded into section 14's **Key
Takeaways**; condensed command list:

| Command | What it drills |
|---|---|
| `kubectl get replicasets` (`kubectl get rs`) | Count/list ReplicaSets |
| `kubectl describe replicaset` or `kubectl get rs -o wide` | Find the image in use |
| `kubectl get rs` columns: `DESIRED` vs `READY` | A mismatch means something's actively broken — check `kubectl describe pods` → `Events` for why |
| Delete one pod owned by a ReplicaSet, re-run `kubectl get pods` | Proves self-healing hands-on |
| `kubectl explain replicaset \| grep VERSION` | Recover the correct `apiVersion` after a creation error |
| Fix a `selector`/template label mismatch | Make `spec.selector.matchLabels` equal `spec.template.metadata.labels` |
| `kubectl delete replicaset replicaset-1 replicaset-2` (or `rs`) | Delete several at once |
| `kubectl edit replicaset new-replica-set` to fix a bad image | Discover existing pods stay broken until deleted/scaled — requires delete+recreate, delete-each-pod, or scale-to-0-then-back |
| `kubectl scale rs new-replica-set --replicas 5` | Final direct scale command |

---

<a id="t16"></a>
## 16. Deployments

> 🎯 **In one sentence:** a Deployment manages a ReplicaSet for you, adding rollout
> history and one-command rollback on top of everything a ReplicaSet already does.

**Concept (in plain terms):** A **Deployment** sits one layer above a ReplicaSet and
is the object you should actually use for stateless applications in practice.
Creating a Deployment **automatically creates a ReplicaSet**, which in turn
**automatically creates Pods** — so the object hierarchy is always exactly:
**Deployment → ReplicaSet → Pods**. What a Deployment adds on top of a bare
ReplicaSet is everything a ReplicaSet lacks: **rollout history, controlled update
strategies, and one-command rollback.**

**Deployment YAML anatomy — structurally identical to a ReplicaSet's:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
  selector:
    matchLabels:
      type: front-end
```

The only difference from a ReplicaSet definition is `kind: Deployment` — every other
field (`replicas`, `selector.matchLabels`, `template`) means exactly the same thing,
because under the hood the Deployment simply generates a ReplicaSet with this same
spec.

**Update strategies (`spec.strategy.type`):**

| Strategy | Behavior |
|---|---|
| `RollingUpdate` (**default**) | Gradually replaces old Pods with new ones a few at a time (tunable via `maxSurge`/`maxUnavailable`) — zero downtime, but both versions run briefly side-by-side. |
| `Recreate` | Kills **all** old Pods first, then creates all new ones — guarantees no two versions run simultaneously, but causes a brief full outage. |

### Diagram

```
 The object hierarchy YOU create is only ever this one chain, top to bottom:

   Deployment "myapp-deployment"      (adds rollout history, strategy, rollback)
          │  auto-creates
          ▼
   ReplicaSet "myapp-deployment-<hash>"   (adds self-healing / replica-count control)
          │  auto-creates
          ▼
   Pod, Pod, Pod                       (runs the actual containers)

 Rolling update (new image applied via set image / edit / apply):

   old ReplicaSet (rs-abc111)  replicas: 3 ──scaled down gradually──► replicas: 0 (KEPT)
   new ReplicaSet (rs-def222)  replicas: 0 ──scaled up gradually  ──► replicas: 3

   kubectl rollout undo simply flips which ReplicaSet is scaled up vs down — only
   possible because the OLD ReplicaSet is kept around at 0 replicas, never deleted.
```

### Worked Example — Creating a Deployment and watching the full chain appear

```bash
kubectl create -f deployment-definition.yaml

kubectl get deployment
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# myapp-deployment   3/3     3            3           10s

kubectl get replicaset
# NAME                          DESIRED   CURRENT   READY
# myapp-deployment-7d8f9c6b5    3         3         3

kubectl get pods
# myapp-deployment-7d8f9c6b5-abcde   Running
# myapp-deployment-7d8f9c6b5-fghij   Running
# myapp-deployment-7d8f9c6b5-klmno   Running

# See the entire chain (plus Services etc.) in one shot
kubectl get all
```

### Worked Example — Imperative creation (fast, exam-friendly)

```bash
kubectl create deployment httpd-frontend --image=httpd:2.4-alpine --replicas=3
kubectl get deployment httpd-frontend
```

### Worked Example — The `kind` case-sensitivity trap

```bash
kubectl create -f deployment-definition-1.yaml
# error: unable to decode ... no kind "deployment" is registered

vi deployment-definition-1.yaml
# kind: deployment   ->   kind: Deployment      (must be capital D)

kubectl create -f deployment-definition-1.yaml
```

### Worked Example — Rolling update and rollback

```bash
kubectl set image deployment/myapp-deployment nginx-container=nginx:1.9.1

kubectl rollout status deployment/myapp-deployment
# Waiting for deployment "myapp-deployment" rollout to finish: 1 out of 3 new replicas updated...
# deployment "myapp-deployment" successfully rolled out

kubectl rollout history deployment/myapp-deployment
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/myapp-deployment nginx-container=nginx:1.9.1

kubectl get rs
# NAME                          DESIRED   CURRENT   READY
# myapp-deployment-7d8f9c6b5    0         0         0     <- old RS, kept, scaled to 0
# myapp-deployment-9a1b2c3d4    3         3         3     <- new RS, scaled up

# Something's wrong with the new image? Roll straight back:
kubectl rollout undo deployment/myapp-deployment
kubectl rollout undo deployment/myapp-deployment --to-revision=1
```

### Worked Example — Choosing a strategy explicitly

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
# vs
spec:
  strategy:
    type: Recreate
```

> ⚠️ **Exam Tip:** after a `kubectl set image`, `kubectl get rs` shows the **old**
> ReplicaSet retained at `0` replicas alongside the **new** one at full replica
> count — that's rollback history working as intended, not a leftover bug. Deleting
> old ReplicaSets manually breaks `kubectl rollout undo`.

### Key Takeaways / Traps

1. A Deployment's YAML is **structurally identical** to a ReplicaSet's
   (`replicas`/`selector`/`template`) — the only difference is `kind: Deployment`;
   it's a wrapper that manages a ReplicaSet for you, not a competing format.
2. `kind` is **case-sensitive** — `deployment` fails validation; it must be exactly
   `Deployment`.
3. Every Deployment update creates a **brand-new ReplicaSet**. `kubectl get rs`
   afterward shows the **old** ReplicaSet retained at `0` replicas (for rollback
   history) alongside the **new** one at full replica count — don't mistake the old
   `0`-replica RS for a leftover bug.
4. `kubectl rollout undo` works *because* that old ReplicaSet was preserved —
   manually deleting old ReplicaSets breaks rollback capability.
5. **`RollingUpdate` is the default strategy** (gradual, zero-downtime, both versions
   briefly coexist); **`Recreate`** kills everything old before creating anything new
   (brief full downtime, but never two versions running together) — pick based on
   whether your app can tolerate mixed versions.
6. `kubectl create deployment <name> --image=<image> --replicas=<n>` is the fast
   imperative path — mirrors `kubectl run` for Pods and `kubectl create -f` +
   `--dry-run=client -o yaml` patterns used elsewhere.
7. `kubectl get all` is the single fastest command to see the whole
   Deployment → ReplicaSet → Pod chain (plus any Services) at once.
8. Both `kubectl describe deployment` and `kubectl get deployment -o wide` surface
   the container image(s) currently in use — useful when you just need to confirm
   what's actually deployed.
9. Since a Deployment auto-manages its ReplicaSet's Pods for you, the "editing the
   image doesn't update running pods" trap from section 14 **does not apply** here —
   this is precisely the problem Deployments solve.

---


<a id="t17"></a>
## 17. Practice Tests - Deployments

> 🎯 **In one sentence:** a hands-on drill of everything in topic 16 — nothing new,
> just practice.

**What this test drills:** counting Pods/ReplicaSets/Deployments across several
states of the cluster, reading the `READY` column, tracing an image back through
`kubectl describe deployment`, diagnosing a not-ready Deployment via Pod events,
fixing a case-sensitivity `kind` typo, and creating a Deployment from scratch (both
declaratively and imperatively). These lessons are already folded into section 16's
**Key Takeaways**; condensed command list:

| Command / Task | What it drills |
|---|---|
| `kubectl get pods` / `kubectl get replicasets` / `kubectl get deployments` | Count objects at each layer of the hierarchy, re-run after each change to see the before/after delta |
| Reading `READY` in `kubectl get pods` | Spot how many of the desired Pods are actually up |
| `kubectl describe deployment` (or `-o wide`) | Find the image used in the Deployment's Pods |
| `kubectl describe pods` → `Events` | Explain *why* a Deployment isn't ready — same habit as Pods/ReplicaSets |
| Fixing `kind: deployment` → `kind: Deployment` | Capitalization trap must be fixed before `kubectl create -f` succeeds |
| Declarative creation from hand-written YAML | See example below |
| Imperative one-liner shortcut | `kubectl create deployment httpd-frontend --image=httpd:2.4-alpine --replicas=3` — explicitly called out as an accepted alternative |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: httpd-frontend
  name: httpd-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: httpd-frontend
  template:
    metadata:
      labels:
        app: httpd-frontend
    spec:
      containers:
      - image: httpd:2.4-alpine
        name: httpd
```

```bash
kubectl create -f my-deployment.yaml
```

---

