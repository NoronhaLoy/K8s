# Kubernetes Scheduling — Complete Notes (with Worked Examples)

> Covers: Manual Scheduling, Labels & Selectors, Taints & Tolerations, Node Selectors,
> Node Affinity, Resource Limits, DaemonSets, Static Pods, Multiple Schedulers,
> and Scheduler Configuration.
>
> Every topic below has: **Concept → YAML/Commands → Worked Example (step by step) →
> Key Takeaways / Traps.**

---

## Big Picture — How All the Pieces Fit Together

```
                              ┌─────────────────────────┐
                              │      kube-scheduler      │
                              │  (Filter -> Score -> Bind)│
                              └─────────────┬─────────────┘
                                            │ decides node, unless...
              ┌─────────────────────────────┼─────────────────────────────┐
              │                             │                             │
   nodeName set manually          schedulerName: my-scheduler     schedulerName unset /
   (Sec 02/03)  -> scheduler          (Sec 18/19) -> a DIFFERENT      default-scheduler
   skipped entirely                   scheduler process/profile       (Sec 20 profiles)
              │                       handles binding instead                │
              ▼                                                              ▼
      kubelet on that node                                     normal Filter/Score pipeline
      starts containers                                        applies these constraints:

                                                          ┌─────────────┬─────────────┬───────────────┐
                                                          │   Taints /  │   Node       │   Resource     │
                                                          │ Tolerations │  Selector /  │  requests /    │
                                                          │  (Sec 06/07)│  Affinity    │  limits        │
                                                          │  node REPELS│ (Sec 08-11)  │  (Sec 12/13)   │
                                                          │  pods       │  pod ATTRACTS│  filters nodes │
                                                          │             │  to node     │  by capacity   │
                                                          └─────────────┴─────────────┴───────────────┘

   Special pod-management controllers that create/manage pods on top of this:
   - DaemonSet (Sec 14/15): one pod per node, uses affinity under the hood
   - Static Pod (Sec 16/17): kubelet creates directly from manifest file, NO scheduler at all

   Underneath everything: Labels & Selectors (Sec 04/05) are the query language every
   mechanism above (nodeSelector, affinity, DaemonSet pod template matching) is built on.
```

---

## 01. Scheduling — Section Introduction

**What is scheduling?**
The kube-scheduler is a control-plane component whose job is to watch for newly created
Pods that have **no node assigned** (`spec.nodeName` is empty) and pick the **best node**
for them to run on.

**How it decides (two phases):**
1. **Filtering** — eliminate nodes that don't meet the Pod's requirements (insufficient
   CPU/memory, taints without matching tolerations, node selector/affinity mismatch, port
   conflicts, etc.).
2. **Scoring** — rank remaining "feasible" nodes using priority functions (least requested
   resources, image locality, pod affinity/anti-affinity spread, etc.) and pick the
   highest-scoring node.

### Diagram — Filter → Score → Bind pipeline

```
        All Nodes in Cluster
      (node01, node02, node03, node04)
                    │
                    ▼
        ┌─────────────────────┐
        │      FILTERING      │   Remove nodes that fail:
        │    (Predicates)     │   - insufficient CPU/memory
        │                     │   - taint not tolerated
        │                     │   - nodeSelector/affinity mismatch
        │                     │   - port conflicts
        └─────────────────────┘
                    │
                    ▼
        Feasible Nodes only
          (node02, node04)
                    │
                    ▼
        ┌─────────────────────┐
        │       SCORING       │   Rank by priority functions:
        │    (Priorities)     │   - least requested resources
        │                     │   - image locality
        │                     │   - affinity/anti-affinity spread
        └─────────────────────┘
                    │
                    ▼
         Highest-scoring node ──────► Bind Pod here
             (node04)
```

### Worked Example — Filtering & Scoring by hand

Suppose you have 3 worker nodes and a new Pod requesting `cpu: 2`, `memory: 4Gi`:

| Node | Allocatable CPU | Allocatable Memory | Already used | Taint? |
|---|---|---|---|---|
| node01 | 4 | 8Gi | 3 CPU, 6Gi | none |
| node02 | 8 | 16Gi | 1 CPU, 2Gi | none |
| node03 | 4 | 8Gi | 1 CPU, 2Gi | `dedicated=gpu:NoSchedule` |

**Filtering step:**
- node01: only 1 CPU / 2Gi free → request needs 2 CPU/4Gi → **fails** filter (insufficient
  resources).
- node02: 7 CPU / 14Gi free → **passes**.
- node03: has a taint the pod doesn't tolerate → **fails** filter.

**Scoring step:** Only node02 remains feasible → it wins by default (no competition).
If node02 and, say, a hypothetical node04 both passed, the scheduler would score both
(e.g., "least requested" prefers the node with more *proportional* free capacity) and pick
the higher score.

**Result:** `kubectl get pod mypod -o wide` → `NODE: node02`.

This manual walk-through is exactly what happens automatically for every Pod you create —
keep this table method in mind whenever debugging a `Pending` pod.

---

## 02. Manual Scheduling

**Concept:** Every Pod has a field `spec.nodeName`. Normally the scheduler sets this. If
you set it yourself **at creation time**, the Pod skips the scheduler entirely and is
picked up directly by the kubelet on that node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: node02      # manually assigned node
```

**Binding object** (what the real scheduler does internally — you can replicate it for an
already-`Pending` pod):
```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node02
```
```bash
curl --header "Content-Type:application/json" --request POST \
  --data '{"apiVersion":"v1","kind":"Binding","metadata":{"name":"nginx"},"target":{"apiVersion":"v1","kind":"Node","name":"node02"}}' \
  http://<api-server>/api/v1/namespaces/default/pods/nginx/binding
```

### Diagram — Normal vs. Manual binding

```
 NORMAL FLOW:
   Pod created (nodeName empty)
          │
          ▼
   kube-scheduler watches ──► picks best node ──► POSTs a Binding object
          │
          ▼
   kubelet on chosen node notices the binding ──► starts containers

 MANUAL FLOW (this topic):
   Pod created WITH nodeName already set in spec
          │
          ▼
   scheduler is skipped entirely (never even looks at this pod)
          │
          ▼
   kubelet on that exact node picks it up directly ──► starts containers
```

### Worked Example — No scheduler running in the cluster

1. Scheduler component is down (check: `kubectl get pods -n kube-system | grep scheduler`
   → not found / `CrashLoopBackOff`).
2. Create a pod normally:
```bash
kubectl run nginx --image=nginx
kubectl get pods -o wide
# STATUS: Pending   NODE: <none>
```
3. Since nothing will ever bind it, fix it manually:
```bash
kubectl get pod nginx -o yaml > nginx.yaml
# edit nginx.yaml, add under spec:
#   nodeName: node01
kubectl delete pod nginx
kubectl apply -f nginx.yaml
kubectl get pods -o wide
# STATUS: Running   NODE: node01
```

**Gotcha proven by this example:** you could **not** just `kubectl edit pod nginx` and add
`nodeName` live — the API server rejects changing `nodeName` on an existing pod, hence the
delete-and-recreate (or Binding API) approach.

---

## 03. Practice Test — Manual Scheduling (Key Takeaways)

### Worked Example — Typical exam-style task
> "A pod called `nginx` is Pending. There is no scheduler in this cluster. Schedule it on
> node `node01` manually."

```bash
kubectl get pods                          # confirm nginx is Pending
kubectl get pods -n kube-system           # confirm no scheduler pod exists
kubectl get pod nginx -o yaml > nginx.yaml
vi nginx.yaml                             # add "nodeName: node01" under spec
kubectl delete pod nginx
kubectl create -f nginx.yaml
kubectl get pod nginx -o wide             # verify NODE column = node01
```

**Other traps to remember:**
- `kubectl get pods -o wide` to see current node (or `<none>` if Pending).
- Can't edit `nodeName` on a running pod — delete & recreate.
- Quick check for scheduler health: `kubectl get pods -n kube-system`.

---

## 04. Labels and Selectors

**Labels** = key/value pairs attached to objects for identification/grouping.
**Selectors** = the way you query objects by their labels.

```yaml
metadata:
  labels:
    app: App1
    function: Front-end
    tier: frontend
    env: prod
```

```bash
kubectl get pods --selector app=App1
kubectl get pods --selector env=prod,tier=frontend   # AND condition
kubectl get all --selector env=prod
```

**Annotations** (non-identifying metadata, not selectable):
```yaml
metadata:
  annotations:
    buildversion: 1.34
```

### Diagram — Labels vs. Selectors

```
   Pods (with labels)                              Selector query
 ┌───────────────────────┐                    ┌──────────────────────────┐
 │ app1-pod               │                    │ kubectl get pods         │
 │  app=App1, env=prod    │ ◄──────────────────┤   -l app=App1            │
 ├───────────────────────┤                     └──────────────────────────┘
 │ app2-pod               │                          matches: app1-pod,
 │  app=App2, env=dev     │   (NOT matched — app=App2)      app3-pod
 ├───────────────────────┤
 │ app3-pod               │
 │  app=App1, env=dev     │ ◄──────────────────  matched — app=App1
 └───────────────────────┘
```

### Worked Example — Labeling and querying a small fleet

```bash
kubectl run app1-pod --image=nginx --labels="app=App1,env=prod,tier=frontend"
kubectl run app2-pod --image=nginx --labels="app=App2,env=dev,tier=backend"
kubectl run app3-pod --image=nginx --labels="app=App1,env=dev,tier=frontend"

kubectl get pods --show-labels
# NAME        LABELS
# app1-pod    app=App1,env=prod,tier=frontend
# app2-pod    app=App2,env=dev,tier=backend
# app3-pod    app=App1,env=dev,tier=frontend

kubectl get pods -l app=App1
# → app1-pod, app3-pod

kubectl get pods -l 'app=App1,env=prod'
# → app1-pod only

kubectl get pods -l 'env in (dev)'
# → app2-pod, app3-pod (set-based selector)
```

**Why this matters for scheduling:** this exact label/selector plumbing is reused by
`nodeSelector`, `nodeAffinity`, Services, and ReplicaSet/Deployment pod management.

---

## 05. Practice Test — Labels and Selectors (Key Takeaways)

### Worked Example
> "How many pods exist with label `env=prod` in namespace `dev`?"
```bash
kubectl get pods -n dev -l env=prod --no-headers | wc -l
```
> "A Deployment `app-deploy` isn't picking up its pods — why?"
```bash
kubectl get deployment app-deploy -o yaml | grep -A3 matchLabels
kubectl get deployment app-deploy -o yaml | grep -A3 "template:" -A6
# Compare spec.selector.matchLabels vs spec.template.metadata.labels — must match exactly.
```
This mismatch (selector ≠ template labels) is the single most common Deployment-creation
exam trap.

---

## 06. Taints and Tolerations

**Analogy:** Taints go on **nodes** — they *repel* pods. Tolerations go on **pods** — they
let a pod ignore a node's taint. One-way relationship: taints never *attract* pods.

```bash
kubectl taint nodes node01 key1=value1:NoSchedule
```
Format: `key=value:effect`.

| Effect | Meaning |
|---|---|
| `NoSchedule` | New non-tolerating pods won't schedule here. Existing pods unaffected. |
| `PreferNoSchedule` | Scheduler tries to avoid, no guarantee. |
| `NoExecute` | New pods won't schedule **and** existing non-tolerating pods get evicted. |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
```

### Diagram — Taint repels, toleration allows back in

```
                     node01  (tainted: app=blue:NoSchedule)
                         ▲
              X blocked  │  plain-pod  (no toleration)
                         │
              ✓ allowed  │  blue-pod   (toleration: app=blue)
                         │

   plain-pod instead lands here (untainted nodes):
        node02  ◄── plain-pod          node03  (also available)
```

### Worked Example — Repel everything except one app

```bash
# Step 1: Taint node01 so nothing schedules there by default
kubectl taint nodes node01 app=blue:NoSchedule

# Step 2: Try a plain pod -> stays Pending on node01, lands elsewhere
kubectl run test-pod --image=nginx
kubectl get pods -o wide   # NODE = node02 or node03, never node01

# Step 3: Pod WITH the matching toleration -> becomes eligible for node01 too
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blue-pod
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
EOF
kubectl get pods -o wide   # blue-pod CAN land on node01 (not guaranteed, just allowed)
```

**Remove the taint:**
```bash
kubectl taint nodes node01 app=blue:NoSchedule-
```

**NoExecute demo (eviction):**
```bash
kubectl taint nodes node01 app=blue:NoExecute
# Any pod already running on node01 WITHOUT a matching toleration is evicted immediately.
```

---

## 07. Practice Test — Taints and Tolerations (Key Takeaways)

### Worked Example
> "Pod `mysql` is stuck Pending. Investigate and fix."
```bash
kubectl describe pod mysql | grep -A5 Events
# Events: "0/3 nodes are available: 3 node(s) had taint {app: blue}, that the pod didn't tolerate."

kubectl describe nodes | grep -i taint
# node01   Taints: app=blue:NoSchedule

# Fix by adding a toleration to the pod (edit + recreate, since pods are immutable here):
kubectl get pod mysql -o yaml > mysql.yaml
# add under spec:
#   tolerations:
#   - key: "app"
#     operator: "Equal"
#     value: "blue"
#     effect: "NoSchedule"
kubectl delete pod mysql
kubectl create -f mysql.yaml
```

**`operator: Exists`** (no value; tolerates any value for that key) — used by system
DaemonSets like `kube-proxy` to run even on tainted/master nodes:
```yaml
tolerations:
- key: "app"
  operator: "Exists"
  effect: "NoSchedule"
```

---

## 08. Node Selectors

**Simplest way to constrain a pod to specific node(s) — single label equality only.**

```bash
kubectl label nodes node01 size=Large
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: data-processor
  nodeSelector:
    size: Large
```

### Diagram — nodeSelector: single exact match only

```
   node01 [size=Large]     node02 [size=Medium]     node03 [size=Small]
        ▲
        │  nodeSelector: { size: Large }  ── exact single-label match only
        │
   large-workload pod
```

### Worked Example

```bash
kubectl label nodes node01 size=Large
kubectl label nodes node02 size=Medium
kubectl label nodes node03 size=Small

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: large-workload
spec:
  containers:
  - name: data-processor
    image: nginx
  nodeSelector:
    size: Large
EOF

kubectl get pod large-workload -o wide
# NODE = node01 (the only node labeled size=Large)
```

**Limitation demonstrated:** if you wanted "Large **OR** Medium", `nodeSelector` cannot
express that — you'd need to give both nodes the *same* label value, or switch to
`nodeAffinity` (next topic) which supports `In [Large, Medium]`.

---

## 09. Node Affinity

**Purpose:** Richer matching than nodeSelector — `In`, `NotIn`, `Exists`, `DoesNotExist`,
`Gt`, `Lt`; and **hard** vs **soft** rules.

| Type | Enforced | Notes |
|---|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard, at scheduling time | Pod stays Pending if unmet |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft, at scheduling time | Best-effort, weight-based |
| `requiredDuringSchedulingRequiredDuringExecution` | Hard, also during execution | Would evict on label change — not GA in most versions |

### Diagram — Required (hard) vs. Preferred (soft) affinity

```
 requiredDuringSchedulingIgnoredDuringExecution  (HARD RULE)

   node01[size=Large]   node02[size=Medium]   node03[size=Small]
        ✓ match              ✓ match               X no match
        └──────────┬──────────┘
                    ▼
        pod may land on EITHER node01 or node02, NEVER node03.
        (If only node03 existed, the pod would stay Pending forever.)

 preferredDuringSchedulingIgnoredDuringExecution  (SOFT RULE)

   node01[size=Large]  ◄── tried first (weight=1)
   node02 / node03     ◄── used as fallback if node01 unavailable

        Pod ALWAYS gets scheduled somewhere — never stuck Pending.
```

### Worked Example — Required (hard) affinity

```bash
kubectl label nodes node01 size=Large
kubectl label nodes node02 size=Medium
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
```
```bash
kubectl apply -f myapp-pod.yaml
kubectl get pod myapp-pod -o wide
# NODE = node01 or node02 (either qualifies; node03 with size=Small would never be picked)
```

### Worked Example — Preferred (soft) affinity, with fallback

```yaml
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: size
            operator: In
            values:
            - Large
```
If **no node** has `size=Large`, the pod is **still scheduled** (just on a less-preferred
node) — unlike the `required` example above, where it would stay `Pending`.

### Worked Example — "IgnoredDuringExecution" proven

```bash
# Pod already Running on node01 (size=Large)
kubectl label nodes node01 size=Large --overwrite
kubectl label nodes node01 size=Small --overwrite   # change the label AFTER pod is running
kubectl get pod myapp-pod -o wide
# STILL Running on node01 — the rule was only checked at scheduling time, not re-evaluated.
```

---

## 10. Practice Test — Node Affinity (Key Takeaways)

### Worked Example
> "Deploy a pod that must run only on nodes labeled `color=blue` or `color=green`."
```bash
kubectl get nodes --show-labels | grep color   # discover which nodes qualify first
```
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: color
          operator: In
          values: ["blue", "green"]
```
If neither label exists on any node, the pod is `Pending` forever — always verify labels
exist **before** troubleshooting the pod spec.

---

## 11. Taints & Tolerations vs. Node Affinity

| Mechanism | Direction | Effect |
|---|---|---|
| Taints & Tolerations | Node → repels pods | Prevents landing unless tolerated; doesn't guarantee placement |
| Node Affinity | Pod → attracts itself | Guarantees pod's own placement; doesn't stop *other* pods from also landing there |

### Diagram — Why you need BOTH mechanisms together

```
                    node03 [dedicated=ml]  (tainted: dedicated=ml:NoSchedule)
                          ▲                        ▲
        taint    X blocks │                        │ ✓ toleration lets
      (keeps            other-pod                  │   ml-training-pod IN
       others OUT)                                   │
                                             affinity ✓ FORCES
                                          ml-training-pod HERE,
                                          nowhere else

   other-pod        (no toleration)         ──X──►  never lands on node03
   ml-training-pod  (toleration+affinity)   ──✓──►  ALWAYS node03, only node03

   Taint alone   -> ml-training-pod could still randomly land on node01/node02
   Affinity alone -> other-pod could still land on node03 (nothing repels it)
   BOTH together  -> node03 is truly dedicated, exclusively, to ml-training-pod
```

### Worked Example — Dedicate a GPU node exclusively to ML pods

```bash
# 1) Taint the GPU node so ordinary pods are repelled
kubectl taint nodes node03 dedicated=ml:NoSchedule
kubectl label nodes node03 dedicated=ml
```
```yaml
# 2) ML pod: tolerate the taint (allowed in) AND require the label (forced in)
apiVersion: v1
kind: Pod
metadata:
  name: ml-training-pod
spec:
  containers:
  - name: ml
    image: tensorflow/tensorflow
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "ml"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: dedicated
            operator: In
            values: ["ml"]
```
**Verify both halves matter:**
- Deploy a normal pod (no toleration) → `kubectl get pods -o wide` → never lands on
  node03 (taint blocks it). ✔ "keeps others OUT"
- Deploy `ml-training-pod` → always lands on node03, never on node01/node02, even though
  it *could* tolerate landing elsewhere (it has no taint there to worry about) because
  affinity forces it. ✔ "keeps ML pods IN"

If you had used **only** the taint: `ml-training-pod` without affinity could still
randomly land on node01/node02 (any node without the taint is still fair game).
If you had used **only** affinity: some unrelated pod could still be scheduled onto
node03 because nothing is repelling it.

---

## 12. Resource Limits (Requests & Limits)

**Requests** = guaranteed amount, used by scheduler to filter feasible nodes.
**Limits** = hard ceiling the container cannot exceed.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "1Gi"
        cpu: 1
      limits:
        memory: "2Gi"
        cpu: 2
```

**CPU:** 1 CPU = 1000m; exceeding limit → **throttled**, not killed.
**Memory:** exceeding limit → **OOMKilled**.

### Diagram — Requests vs. Limits on a node

```
 Node capacity:   |------------------ 4 CPU / 8Gi allocatable ------------------|

 Container:       |-- request: 1 CPU/1Gi --|---- headroom up to limit: 2 CPU/2Gi ----|
                    ▲ guaranteed floor;       ▲ container MAY burst up to here;
                      scheduler uses this        exceeding CPU limit -> throttled
                      to filter feasible          exceeding memory limit -> OOMKilled
                      nodes
```

### Worked Example — Requests filtering out a node

```bash
kubectl describe node node01 | grep -A5 Allocatable
# cpu: 2, memory: 2Gi   (already fully used by other pods)
```
```yaml
resources:
  requests:
    cpu: "1"
    memory: "1Gi"
```
```bash
kubectl apply -f big-pod.yaml
kubectl get pod big-pod
# Pending
kubectl describe pod big-pod | grep -A3 Events
# "0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory."
```

### Worked Example — Memory limit → OOMKilled

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
spec:
  containers:
  - name: memory-hog
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```
```bash
kubectl apply -f memory-hog.yaml
kubectl get pod memory-hog
# NAME          READY   STATUS      RESTARTS
# memory-hog    0/1     OOMKilled   3
```

### Worked Example — LimitRange & ResourceQuota in a namespace

```bash
kubectl create namespace team-a
```
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
  namespace: team-a
spec:
  limits:
  - default:
      cpu: 500m
    defaultRequest:
      cpu: 500m
    max:
      cpu: "1"
    min:
      cpu: 100m
    type: Container
```
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```
```bash
kubectl apply -f limitrange.yaml
kubectl apply -f quota.yaml
kubectl run test --image=nginx -n team-a
kubectl get pod test -n team-a -o yaml | grep -A4 resources
# requests.cpu/limits.cpu auto-filled to 500m from the LimitRange default, since the pod spec omitted resources.
```

---

## 13. Practice Test — Resource Limits (Key Takeaways)

### Worked Example
> "Pod `webapp` keeps restarting. Diagnose."
```bash
kubectl describe pod webapp | grep -i "Last State" -A3
# Last State:   Terminated
#   Reason:     OOMKilled
#   Exit Code:  137
```
Fix: raise `resources.limits.memory` in the pod/deployment spec, then:
```bash
kubectl get pod webapp -o yaml > webapp.yaml   # edit limits
kubectl delete pod webapp
kubectl create -f webapp.yaml
# (if owned by a Deployment instead: kubectl edit deployment webapp)
```
`kubectl top pod` / `kubectl top node` (needs metrics-server) to compare live usage vs.
requested/limited values.

---

## 14. DaemonSets

**Purpose:** Exactly one pod per node (or per selected subset), auto-added on new nodes,
auto-removed with the node. Examples: `kube-proxy`, CNI plugins, node-exporter, fluentd.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
  labels:
    app: monitoring-agent
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: monitoring-agent
        image: monitoring-agent
```

### Diagram — DaemonSet: one pod per node, always

```
    node01            node02            node03            node04 (joins later)
      │                 │                 │                    │
      ▼                 ▼                 ▼                    ▼
 [monitoring]      [monitoring]      [monitoring]      [monitoring]  ◄── auto-created
     pod               pod                pod               pod          on node join

  DaemonSet controller continuously ensures exactly 1 matching pod per node.
  Node removed  -> its pod is garbage collected automatically.
```

### Worked Example — Deploy and verify one-per-node

```bash
kubectl get nodes
# node01, node02, node03   (3 nodes)

kubectl apply -f monitoring-daemonset.yaml
kubectl get daemonset monitoring-daemon
# DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# 3         3         3       3            3

kubectl get pods -o wide -l app=monitoring-agent
# monitoring-daemon-abcde   node01
# monitoring-daemon-fghij   node02
# monitoring-daemon-klmno   node03
```

### Worked Example — Restrict a DaemonSet to a subset of nodes

```bash
kubectl label nodes node01 disk=ssd
```
```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: monitoring-agent
        image: monitoring-agent
```
```bash
kubectl apply -f monitoring-daemonset.yaml
kubectl get pods -o wide -l app=monitoring-agent
# Only 1 pod, on node01
```

**Internals note:** modern Kubernetes schedules DaemonSet pods via the **default
scheduler using auto-injected node affinity** (not raw `nodeName` bypass like very old
versions) — so it still respects taints/resources like a normal pod.

---

## 15. Practice Test — DaemonSets (Key Takeaways)

### Worked Example
> "Convert this existing Deployment YAML into a DaemonSet named `fluentd`."
```bash
kubectl get deployment webapp -o yaml > webapp.yaml
# copy spec.selector and spec.template into a new file, set:
#   apiVersion: apps/v1
#   kind: DaemonSet
# remove: replicas, strategy
kubectl apply -f fluentd-daemonset.yaml
kubectl get pods -o wide | grep fluentd
# one pod per matching node
```
Verify count: `kubectl get pods -l name=fluentd --no-headers | wc -l` should equal
`kubectl get nodes --no-headers | wc -l` (minus any tainted/excluded nodes).

---

## 16. Static Pods

**Concept:** Pods created directly by the **kubelet** from files in a manifest directory —
no API server or scheduler involved.

```bash
ps -ef | grep kubelet | grep -- --config
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# staticPodPath: /etc/kubernetes/manifests
```

### Diagram — Static pod lifecycle (no scheduler, no API server needed)

```
  Node filesystem                      kubelet                    API server (if joined)
 /etc/kubernetes/manifests/  ──watches──► reads file  ──creates──►  "mirror pod"
   static-busybox.yaml                    & starts container       (read-only reflection,
                                                                     visible via kubectl)

  kubectl delete pod  ──X──►  mirror pod removed, but kubelet immediately
                               RECREATES it from the manifest file still on disk!

  rm the manifest file on the node  ──✓──►  kubelet detects removal,
                                             stops & removes the container for real
```

### Worked Example — Create a static pod manually

```bash
# SSH onto the node
ssh node01

# Write a pod definition into the static manifest directory
cat <<EOF > /etc/kubernetes/manifests/static-busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-busybox
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
EOF

# Kubelet auto-detects and creates it within seconds
crictl ps | grep busybox
```
Back on the control plane (if this node is joined to a cluster):
```bash
kubectl get pods -o wide
# static-busybox-node01   Running   node01     <- mirror pod, name suffixed with node hostname
```

### Worked Example — Why kubectl delete doesn't work

```bash
kubectl delete pod static-busybox-node01
# pod "static-busybox-node01" deleted
kubectl get pods
# static-busybox-node01 reappears within seconds — kubelet recreates it from the manifest file!
```
To truly remove it:
```bash
ssh node01
rm /etc/kubernetes/manifests/static-busybox.yaml
# kubelet detects removal and deletes the pod for real
```

### Worked Example — Why kubeadm uses static pods for the control plane

```bash
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
kubectl get pods -n kube-system
# kube-apiserver-controlplane   Running
# kube-scheduler-controlplane   Running
```
This solves the chicken-and-egg problem: the API server/scheduler need *something* to run
them before the API server itself exists — kubelet + static manifest files does that job.

---

## 17. Practice Test — Static Pods (Key Takeaways)

### Worked Example
> "There's a static pod `kube-apiserver-node01` failing. Fix its config and confirm it
> restarts automatically."
```bash
ssh node01
vi /etc/kubernetes/manifests/kube-apiserver.yaml   # fix the bad flag/image
# save -> kubelet detects the file change and recreates the pod automatically, no kubectl needed
crictl ps | grep apiserver
```
> "How do you tell if pod `X` is static vs. controller-managed?"
```bash
kubectl get pod X -o yaml | grep -i ownerReferences -A3
# Static/mirror pods typically show no Deployment/ReplicaSet owner; name ends in -<hostname>.
```

---

## 18. Multiple Schedulers

**Why:** Run the default scheduler **plus** custom scheduler(s) with their own logic; each
pod picks its scheduler via `schedulerName`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --scheduler-name=my-scheduler
    image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
    name: kube-scheduler
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  schedulerName: my-scheduler
```

### Diagram — Two schedulers watching different pods

```
       default-scheduler                        my-scheduler (deployed separately)
             │                                          │
   watches pods with                          watches pods with
   schedulerName unset / "default-scheduler"  schedulerName: "my-scheduler"
             │                                          │
             ▼                                          ▼
     Filter -> Score -> Bind                    custom Filter/Score/Bind logic
     (uses default logic)                       (whatever the custom binary does)

   Pod with schedulerName: "my-scheduler" but NO my-scheduler pod running:
             ──► nobody is watching for it ──► stays Pending forever
```

### Worked Example — Deploy custom scheduler and verify placement

```bash
kubectl apply -f my-custom-scheduler.yaml -n kube-system
kubectl get pods -n kube-system | grep my-custom-scheduler
# my-custom-scheduler   Running

kubectl apply -f nginx-custom-scheduled.yaml
kubectl get pods -o wide
# nginx   Running   node02

kubectl get events -o wide | grep nginx
# ... Scheduled ...  my-scheduler   Successfully assigned default/nginx to node02
```

### Worked Example — Forgotten/broken custom scheduler

```bash
kubectl get pods -n kube-system | grep my-scheduler
# (nothing — it's not running)
kubectl get pods
# nginx   Pending   <none>
kubectl describe pod nginx | grep -A5 Events
# (no "Scheduled" event at all — nothing is watching for schedulerName=my-scheduler)
```
Fix: redeploy the custom scheduler pod, then the pending pod gets picked up automatically.

---

## 19. Practice Test — Multiple Schedulers (Key Takeaways)

### Worked Example
> "Pod `nginx` uses `schedulerName: my-scheduler` and is stuck Pending. Find out why and
> fix it."
```bash
kubectl get pods -n kube-system            # is a pod/deployment named my-scheduler even running?
kubectl logs my-scheduler -n kube-system   # check for crash/config errors, e.g. wrong kubeconfig path
kubectl describe pod nginx                 # confirm no Scheduled event
# Fix the scheduler pod's config (e.g. correct --scheduler-name or --kubeconfig), reapply,
# then confirm nginx transitions Pending -> Running.
```

---

## 20. Configuring Kubernetes Schedulers

Modern approach: **KubeSchedulerConfiguration** file + **scheduler profiles**, replacing
old CLI-flag-per-behavior and scheduler-policy JSON.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: my-scheduler-2
leaderElection:
  leaderElect: true
```
Run with: `--config=/etc/kubernetes/my-scheduler-config.yaml`.

**Extension-point pipeline:**
`Sort → PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit →
PreBind → Bind → PostBind`

### Diagram — Scheduling framework extension points (one process, many profiles)

```
 Sort → PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit → PreBind → Bind → PostBind
  │         │           │         │           │         │        │         │        │        │       │
 queue    quick      full node  handle all-   prep     rank    hold      wait/    setup    actually  cleanup
 order    checks     eligibility filtered-out  score    feasible resources approve  volumes   bind pod  / notify
                                  case          inputs   nodes    (reserve) (gang    etc.      to node
                                                                             sched.)

   default-scheduler profile:        runs the FULL pipeline above
   no-scoring-scheduler profile:     PreScore + Score stages DISABLED
                                      -> first feasible node from Filter wins, no ranking
   (Both profiles run inside the SAME kube-scheduler binary/process.)
```

### Worked Example — Two profiles in one scheduler process

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: no-scoring-scheduler
    plugins:
      preScore:
        disabled:
        - name: '*'
      score:
        disabled:
        - name: '*'
```
```bash
# static pod manifest for kube-scheduler references this config:
cat /etc/kubernetes/manifests/kube-scheduler.yaml | grep -- --config

# a pod opts into the lightweight profile just by name, same binary handles both:
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fast-schedule-pod
spec:
  schedulerName: no-scoring-scheduler
  containers:
  - name: nginx
    image: nginx
```
```bash
kubectl apply -f fast-schedule-pod.yaml
kubectl get events | grep fast-schedule-pod
# Scheduled by no-scoring-scheduler — first feasible node chosen, no scoring pass run.
```

### Worked Example — Inspecting a live kubeadm cluster's scheduler config

```bash
kubectl get pods -n kube-system | grep scheduler
kubectl describe pod kube-scheduler-controlplane -n kube-system | grep -- --config
cat /etc/kubernetes/manifests/kube-scheduler.yaml
# confirm --config flag path, or absence of it (meaning pure CLI-flag config, older style)
```

**Key distinction:**
- Separate scheduler **binaries/pods** (topic 18) = simplest, fully independent, heavier.
- Scheduler **profiles** in one process (this topic) = lightweight, modern, preferred for
  most customization needs.

---

## 21. Download Presentation Deck

No YAML here — it's the official slide deck download for this section. Treat it as a
**visual example bank**: use it to see diagrammed versions of the exact scenarios above.

### How to use it effectively (worked checklist)

1. **Filtering/Scoring diagram** — pair with the Section 01 table example above; the
   slides show it as a funnel (all nodes → feasible nodes → scored/ranked → winner).
2. **Taint/Toleration diagram** — cross-check against the Section 06/07 examples; slides
   typically show the "repel" arrow direction, which helps cement why taints alone never
   *guarantee* placement.
3. **Node Affinity diagram** — compare the required-vs-preferred boxes in the deck with
   the Section 09 required/preferred YAML pair above.
4. **Static Pod vs. DaemonSet diagram** — use it to visually confirm the comparison table
   in Section 16.
5. Save a local copy (`kubernetes-scheduling-slides.pdf`) alongside this notes file so
   diagrams and worked examples are cross-referenced during review.

---

## Quick-Reference Cheat Sheet

| Goal | Mechanism |
|---|---|
| Bypass scheduler entirely | `nodeName` in pod spec, or static pod |
| Repel pods from a node | Taint the node + require matching toleration |
| Simple pin pod → node | `nodeSelector` (single label match only) |
| Complex pin pod → node(s) | `nodeAffinity` (In/NotIn/Exists/Gt/Lt, soft or hard) |
| Dedicate node to one workload exclusively | Taint (keep others out) **+** Node Affinity (guarantee target lands there) |
| Guarantee minimum resources / filter nodes by capacity | `resources.requests` |
| Cap resource usage | `resources.limits` (CPU throttled, memory OOMKilled) |
| Namespace-wide default/max resources | `LimitRange` |
| Namespace-wide total resource cap | `ResourceQuota` |
| One pod per node automatically | `DaemonSet` |
| Control-plane / node-level pods without API server | `Static Pod` (kubelet manifest path) |
| Custom scheduling logic, fully separate | Additional scheduler binary + `schedulerName` |
| Custom scheduling logic, lightweight | Scheduler **profiles** in `KubeSchedulerConfiguration` |

---

## Common Exam/Practice Traps to Remember

1. `nodeName` and `affinity`/`tolerations` **cannot be edited on a live pod** — always
   delete & recreate (or edit the owning Deployment).
2. Taint effect **`NoExecute`** evicts pods that are *already running* and lack the
   toleration — the only effect that acts on existing pods.
3. `nodeSelector` = AND-only, single label; need `nodeAffinity` for OR/NOT/Exists logic.
4. Taints ⇒ repel (node's perspective); Affinity ⇒ attract (pod's perspective). Combine
   both to truly dedicate a node (see Section 11 worked example).
5. Memory limit exceeded → `OOMKilled` (exit code 137). CPU limit exceeded → throttled,
   not killed.
6. DaemonSet pods are (in modern k8s) scheduled via auto-injected node affinity, not by
   bypassing the scheduler — don't claim "DaemonSets bypass the scheduler" as an absolute.
7. Static pods show up in `kubectl get pods` as **mirror pods** but must be
   created/deleted via the **node's filesystem**, not via `kubectl delete` (see Section 16
   worked example — deleted pod reappears).
8. A pod with `schedulerName: foo` where no scheduler named `foo` exists/runs stays
   `Pending` indefinitely — always verify the custom scheduler pod is `Running` first.
9. LimitRange/ResourceQuota only affects **new** pod creations, not retroactive to
   existing pods.
10. Control plane components (`kube-apiserver`, `etcd`, `kube-scheduler`,
    `kube-controller-manager`) in kubeadm clusters are **static pods** — check
    `/etc/kubernetes/manifests/` on the control-plane node to troubleshoot them.

---

*End of notes — every topic now includes at least one worked, runnable example. Good luck
with practice tests!*
