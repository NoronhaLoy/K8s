
---

<a id="t18"></a>
## 18. Namespaces

> 🎯 **In one sentence:** a Namespace is a virtual sub-cluster used to isolate
> teams/projects on one physical cluster — and a handful of resource types (Nodes,
> PVs, ...) live outside all namespaces entirely.

**Concept (in plain terms):** A **Namespace** is a way to divide a single physical
cluster into multiple **virtual clusters** — logical groupings used for multi-team or
multi-project isolation. Every object you have created so far (Pods, Deployments,
Services) has always lived inside *some* namespace, even when you never typed one —
by default, that's the **`default`** namespace, automatically created when the
cluster is first set up.

**Namespaces created automatically on every cluster:**

| Namespace | Purpose |
|---|---|
| `default` | Where objects land if you don't specify a namespace |
| `kube-system` | Cluster-internal objects — control-plane static pods, kube-proxy, CoreDNS, etc. Don't touch/create things here casually. |
| `kube-public` | Resources meant to be readable by **all** users, even unauthenticated ones (e.g. a `cluster-info` ConfigMap) |
| `kube-node-lease` | Holds **Lease** objects, one per node, used for node heartbeats (a lighter-weight mechanism than updating the full Node object on every heartbeat) |

**Why namespaces exist:** they let separate teams/projects share one physical cluster
without naming collisions or accidental interference, and let you apply policies
**per namespace** — resource quotas, RBAC roles/role-bindings, and network policies
can all be scoped to a namespace.

**Namespaced vs. cluster-scoped resources:** most objects (Pods, ReplicaSets,
Deployments, Services, ConfigMaps, Secrets, PVCs, Roles/RoleBindings, ...) live
*inside* a namespace. A handful of resources are **cluster-scoped** — they exist once,
globally, and are never namespaced: **Nodes**, **PersistentVolumes**,
**StorageClasses**, **ClusterRoles/ClusterRoleBindings**,
**CertificateSigningRequests**, and **Namespaces themselves**.

```bash
# Ask kubectl directly instead of memorizing the list:
kubectl api-resources --namespaced=true    # pods, deployments, services, configmaps...
kubectl api-resources --namespaced=false   # nodes, persistentvolumes, namespaces,
                                            # clusterroles, clusterrolebindings, ...
```

**Cross-namespace DNS:** every Service gets a DNS entry automatically, of the form
`<service-name>.<namespace>.svc.cluster.local`. A Pod talking to a Service in its
**own** namespace can use just the short name (`db-service`); a Pod talking to a
Service in a **different** namespace must include at least the namespace
(`db-service.dev`), or the fully-qualified form
(`db-service.dev.svc.cluster.local`).

### Diagram

```
                         PHYSICAL CLUSTER
   ┌──────────────────────────────────────────────────────────────────────┐
   │  Namespace: default        Namespace: dev          Namespace: prod    │
   │  ┌─────────────────┐      ┌─────────────────┐      ┌───────────────┐ │
   │  │ myapp-pod         │      │ myapp-pod         │      │ myapp-pod      │ │
   │  │ (same name is OK, │      │ (same name is OK, │      │                │ │
   │  │  different NS)    │      │  different NS)    │      │                │ │
   │  └─────────────────┘      └─────────────────┘      └───────────────┘ │
   │                                                                        │
   │  Namespace: kube-system     Namespace: kube-public   Namespace:        │
   │  (control-plane pods,       (cluster-info, readable  kube-node-lease  │
   │   kube-proxy, CoreDNS)       by anyone)                (node heartbeat │
   │                                                          Lease objects)│
   └──────────────────────────────────────────────────────────────────────┘

   Cluster-scoped (NOT inside any namespace, exist exactly once):
   Nodes · PersistentVolumes · StorageClasses · ClusterRoles ·
   ClusterRoleBindings · CertificateSigningRequests · Namespaces themselves

   DNS resolution across namespaces:
     Pod in "marketing" NS  ──► db-service                          (same NS, short name)
     Pod in "dev" NS        ──► db-service.marketing                (cross-NS, needs NS)
     Anyone, anywhere       ──► db-service.marketing.svc.cluster.local  (full FQDN, always works)
```

### Worked Example — Creating and using namespaces

```yaml
# namespace-dev.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```bash
kubectl create -f namespace-dev.yaml
# — or, the faster imperative equivalent —
kubectl create namespace dev

kubectl get namespace
# NAME              STATUS   AGE
# default           Active   30d
# kube-node-lease   Active   30d
# kube-public       Active   30d
# kube-system       Active   30d
# dev               Active   5s

# List pods in a non-default namespace explicitly
kubectl get pods --namespace=kube-system

# Create a pod-definition's object inside a specific namespace via the CLI flag
kubectl create -f pod-definition.yaml --namespace=dev
```

```yaml
# ...or pin the namespace permanently into the manifest itself, so it always lands
# in "dev" even if someone forgets the --namespace flag later:
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev
  labels:
    app: myapp
    type: front-end
spec:
  containers:
  - name: nginx-container
    image: nginx
```

### Worked Example — Switching your active namespace permanently

```bash
# Every kubectl command defaults to whatever namespace is set in your current context.
# Rather than typing --namespace=dev on every single command, change the context itself:
kubectl config set-context $(kubectl config current-context) --namespace=dev
# — equivalently, on newer kubectl —
kubectl config set-context --current --namespace=dev

# Confirm it stuck — new pods now land in "dev" with no flag needed:
kubectl config view --minify | grep namespace:
#     namespace: dev

kubectl run nginx --image=nginx     # lands in "dev", not "default"
```

### Worked Example — Seeing everything, everywhere, and limiting it with a quota

```bash
# See pods across ALL namespaces at once (essential — kubectl get pods alone only
# shows your current namespace's pods)
kubectl get pods --all-namespaces
# NAMESPACE   NAME              READY   STATUS
# default     myapp-pod         1/1     Running
# dev         myapp-pod         1/1     Running
# marketing   blue              1/1     Running
```

```yaml
# compute-quota.yaml — cap how much a namespace's occupants can collectively consume
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```

```bash
kubectl create -f compute-quota.yaml
kubectl describe resourcequota compute-quota -n dev
# Resource         Used  Hard
# --------         ----  ----
# limits.cpu       0     10
# limits.memory    0     10Gi
# pods             0     10
# requests.cpu     0     4
# requests.memory  0     5Gi
```

> ⚠️ **Exam Tip:** cross-namespace DNS is one of the most-tested facts in this whole
> module — same-namespace traffic can use the bare service name, but cross-namespace
> traffic needs at least `<service>.<namespace>`, and the fully-qualified
> `<service>.<namespace>.svc.cluster.local` always works.

### Key Takeaways / Traps

1. Objects always live in **some** namespace (even `default`, silently) — never assume
   `kubectl get pods` with no flags shows the *whole* cluster; use
   `kubectl get pods --all-namespaces` (or `-A`) for that.
2. The four built-in namespaces to recognize by name: **`default`**,
   **`kube-system`** (control-plane/system pods — leave alone), **`kube-public`**
   (world-readable), **`kube-node-lease`** (node heartbeat Lease objects).
3. A handful of resources are **cluster-scoped, never namespaced** — the classic exam
   trap list: **Nodes, PersistentVolumes, StorageClasses, ClusterRoles/
   ClusterRoleBindings, CertificateSigningRequests, Namespaces** themselves. Use
   `kubectl api-resources --namespaced=false` if unsure rather than guessing.
4. **DNS across namespaces** — same-namespace traffic only needs the bare service
   name (`db-service`); cross-namespace traffic needs at least
   `<service>.<namespace>` and, to be unambiguous/fully correct, the full form
   `<service>.<namespace>.svc.cluster.local`. This is one of the single most-tested
   networking facts in this whole module.
5. `kubectl config set-context --current --namespace=<ns>` changes your **shell's
   default namespace going forward** — it does **not** move any existing objects; it
   just saves you typing `--namespace=` on every subsequent command.
6. `kubectl create namespace <name>` (imperative) is equivalent to writing a
   `kind: Namespace` YAML and `kubectl create -f`-ing it — use whichever is faster.
7. `kubectl run redis --image=redis --namespace=finance` and
   `kubectl get pods --all-namespaces | grep blue` are the two fastest patterns for,
   respectively, "create something in namespace X" and "which namespace has object Y"
   — both come straight out of the practice-test format and reappear constantly.
8. A **ResourceQuota** is itself a namespaced object (`metadata.namespace` must be
   set) — it only constrains *its own* namespace, and does nothing until objects
   inside that namespace start requesting/using resources against its `hard` limits.

---

<a id="t19"></a>
## 19. Practice Test - Namespaces

> 🎯 **In one sentence:** a hands-on drill of everything in topic 18 — nothing new,
> just practice.

Every lesson this test exercises is already folded into topic 18's **Key Takeaways**
above — the practice work is simply: counting pods per-namespace vs. cluster-wide
(`kubectl get pods -n <ns>` vs `--all-namespaces`), creating objects directly into a
named namespace (`--namespace=` flag or a pinned `metadata.namespace` field),
switching your active namespace context, and correctly identifying which resource
types are cluster-scoped rather than namespaced.

---

<a id="t20"></a>
## 20. Services

> 🎯 **In one sentence:** a Service is a stable virtual IP + DNS name in front of a
> changing set of Pods — the fix for "Pod IPs change every time a Pod restarts."

**Concept (in plain terms):** A **Service** is a Kubernetes object that provides a
stable, single point of network access to a group of Pods — solving the problem that
Pods are ephemeral and their IPs change every time they're recreated. Services enable
communication **between** components inside the cluster, and (for some types) **into**
the cluster from the outside world.

**Why you can't just use Pod IPs directly:** a Pod running a web server on the node
itself is reachable at `<nodeIP>:<podPort>` **from that node**, but an external user
hitting the node's IP directly has nothing routing them to the container's port — and
even if it did, that Pod could be deleted/rescheduled to a different IP/node at any
time. A Service sits in front of the Pods and gives external and internal clients a
single, stable address that survives individual Pods coming and going.

**The three Service types:**

| Type | Scope | Behavior |
|---|---|---|
| **ClusterIP** (default if `type` omitted) | Internal only | Creates a stable virtual IP reachable **only from inside the cluster** — used for service-to-service communication (e.g. frontend tier talking to backend tier). |
| **NodePort** | Internal + external | Everything ClusterIP does, **plus** opens the *same* port on **every node** in the cluster, so external traffic to `<any-node-IP>:<nodePort>` reaches the Service. |
| **LoadBalancer** | Internal + external | Everything NodePort does, **plus** provisions an actual cloud load balancer (AWS ELB, GCP LB, etc.) in front of it, on **supported cloud providers only** — on bare-metal/unsupported providers it just behaves like NodePort. |

**Service YAML anatomy and the three port fields — the single most-confused part of
this topic:**

| Field | Meaning |
|---|---|
| `spec.type` | `ClusterIP` (default) / `NodePort` / `LoadBalancer` |
| `spec.ports[].port` | The port the **Service itself** listens on (what other pods/clients hit) |
| `spec.ports[].targetPort` | The port on the **backend Pod/container** traffic gets forwarded to (defaults to same as `port` if omitted) |
| `spec.ports[].nodePort` | (NodePort/LoadBalancer only) The port opened on **every node's** own IP; valid range **30000–32767**; auto-assigned from that range if omitted |
| `spec.selector` | Label(s) used to find which Pods this Service sends traffic to — same `key: value` matching mechanics as ReplicaSets/Deployments |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
  - targetPort: 80    # port the container is listening on
    port: 80           # port the Service exposes internally
    nodePort: 30008    # port opened on every node (must be 30000-32767)
  selector:
    app: myapp
    type: front-end
```

```bash
kubectl create -f service-definition.yaml
kubectl get services
# NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# myapp-service    NodePort   10.106.1.100   <none>        80:30008/TCP   5s

# From outside the cluster (e.g. your laptop), reach it via ANY node's IP:
curl http://192.168.1.2:30008
```

### Diagram — NodePort across multiple nodes, and multiple backend Pods

```
   EXTERNAL USER ──► curl http://<any-node-IP>:30008
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   ┌───────────┐        ┌───────────┐        ┌───────────┐
   │  node01    │        │  node02    │        │  node03    │
   │ :30008     │        │ :30008     │        │ :30008     │   <- SAME nodePort
   │ (kube-proxy│        │ (kube-proxy│        │ (kube-proxy│      opened on
   │  rule)     │        │  rule)     │        │  rule)     │      EVERY node,
   └─────┬─────┘        └─────┬─────┘        └─────┬─────┘      even ones with
         │                      │                      │           no matching
         └──────────────────────┼──────────────────────┘           pod at all!
                                 ▼
                  Service "myapp-service" (selector: app=myapp)
                        picks ONE matching backend Pod
                                 │
             ┌───────────────────┼───────────────────┐
             ▼                   ▼                   ▼
       Pod (node01)        Pod (node02)        Pod (node03)
       app=myapp            app=myapp            app=myapp

   The Service's Endpoints object lists every matching Pod's IP:targetPort.
   kube-proxy load-balances (random, by default) across ALL of them — regardless
   of which node you happened to connect through.
```

### Worked Example — Connecting a Service to Pods via `selector`, then load-balancing across replicas

```yaml
# service-definition.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
  - targetPort: 80
    port: 80
    nodePort: 30008
  selector:          # must match the Pod template's labels exactly, same
    app: myapp        # matching rules as ReplicaSet/Deployment selectors
    type: front-end
```

```bash
kubectl create -f service-definition.yaml
kubectl get pods -l app=myapp,type=front-end
# myapp-pod-1   Running
# myapp-pod-2   Running
# myapp-pod-3   Running

kubectl describe service myapp-service
# Selector:          app=myapp,type=front-end
# Endpoints:         10.244.1.5:80,10.244.2.7:80,10.244.3.9:80
#                    ^ one entry per matching Pod — this IS the load-balancing pool

# Repeated calls land on different backend pods (random selection by kube-proxy):
for i in 1 2 3 4 5; do curl -s http://192.168.1.2:30008 | grep Hostname; done
```

> ⚠️ **Exam Tip:** memorize `port` vs `targetPort` vs `nodePort` cold — `port` is what
> the Service itself listens on, `targetPort` is the container's actual port, and
> `nodePort` (30000–32767 only) is the externally-reachable port on every node.

### Key Takeaways / Traps

1. `type: ClusterIP` is the **default** if `spec.type` is omitted entirely — don't
   assume a Service with no explicit type does nothing; it's just internal-only.
2. **`port` vs `targetPort` vs `nodePort`** — the classic mix-up: `port` is what the
   Service listens on internally, `targetPort` is the container's actual port, and
   `nodePort` (NodePort/LoadBalancer only) is the externally-reachable port on
   every node. If `targetPort` is omitted it defaults to `port`'s value.
3. **`nodePort` must fall in 30000–32767** — anything outside that range is rejected
   at creation time; if you don't specify one, Kubernetes auto-assigns a free port
   in that range for you.
4. A NodePort Service opens its port on **every node**, not just the node(s) actually
   running a matching Pod — you can reach the app through *any* node's IP.
5. A Service with an empty `Endpoints` list (`kubectl describe service` →
   `Endpoints: <none>`) almost always means `spec.selector` doesn't match any Pod's
   `labels` — same class of bug as ReplicaSet selector mismatches, and it is a
   Service/label problem, **not** a kube-proxy problem (see topic 10).
6. `LoadBalancer` only actually provisions a cloud load balancer on a **supported
   cloud provider** (AWS/GCP/Azure/etc.) — on bare-metal or local lab clusters it
   behaves exactly like `NodePort` (an `EXTERNAL-IP` will show `<pending>` forever).
7. `kubectl get services` (`kubectl get svc`) shows `TYPE`, `CLUSTER-IP`,
   `EXTERNAL-IP`, and the combined `PORT(S)` column as `port:nodePort/protocol` for
   NodePort Services — read it left-to-right rather than guessing which number is
   which.
8. `kubectl describe service <name>` surfaces `Selector`, `Endpoints`, and `Labels`
   directly — the fastest way to confirm targetPort, endpoint count, or label count
   without parsing raw YAML (a direct callout from the Services practice test).

---

<a id="t21"></a>
## 21. Services - ClusterIP

> 🎯 **In one sentence:** ClusterIP is the default, internal-only Service type — the
> right choice for tier-to-tier traffic that should never be reachable from outside
> the cluster.

**Concept (in plain terms):** `ClusterIP` is the Service type used for
**internal-only** communication between tiers of an application — e.g. a set of
frontend Pods calling a set of backend Pods — without ever needing to know individual
Pod IPs, which change every time a Pod is recreated. The Service creates a **virtual
IP** inside the cluster that stays constant regardless of which/how-many Pods are
actually backing it at any moment, and it is the **default** Service type when
`spec.type` is left unset.

**The core problem it solves:** with 3 replicas of a backend tier spread across
multiple nodes, a frontend Pod can't reliably hard-code any one backend Pod's IP —
Pods are recreated with new IPs constantly. A ClusterIP Service groups all matching
backend Pods behind one unchanging internal address+DNS name, and load-balances
across whichever Pods currently match its `selector`.

```yaml
# service-definition.yaml
apiVersion: v1
kind: Service
metadata:
  name: back-end
spec:
  type: ClusterIP        # default — could be omitted entirely with the same effect
  ports:
  - targetPort: 80         # port on the backend container
    port: 80                 # port the Service itself exposes internally
  selector:
    app: myapp
    type: back-end
```

```bash
kubectl create -f service-definition.yaml
kubectl get services
# NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# back-end   ClusterIP   10.107.34.121   <none>        80/TCP    4s
```

### Diagram — Frontend tier reaching backend tier through one stable virtual IP

```
   Frontend Pod-1 ──┐
   Frontend Pod-2 ──┼──► curl http://back-end:80    (Service DNS name, same namespace)
   Frontend Pod-3 ──┘             │
                                   ▼
                    Service "back-end"  (ClusterIP: 10.107.34.121)
                    spec.selector: {app: myapp, type: back-end}
                                   │
                    Endpoints = every Pod matching that selector, right now:
             ┌─────────────────────┼─────────────────────┐
             ▼                     ▼                     ▼
      Backend Pod-1          Backend Pod-2          Backend Pod-3
      (node01)                (node02)                (node03)

   If Backend Pod-2 dies and is replaced with a new IP, the Service's Endpoints list
   updates automatically — frontend Pods never notice, since they only ever talk to
   the unchanging Service name/ClusterIP, never to Pod IPs directly.
```

### Worked Example — Grouping a backend tier and testing it from the frontend

```bash
kubectl create -f service-definition.yaml
kubectl describe service back-end
# Type:              ClusterIP
# IP:                10.107.34.121
# Port:              <unset>  80/TCP
# TargetPort:        80/TCP
# Endpoints:         10.244.1.5:80,10.244.2.7:80,10.244.3.9:80

# From any Pod in the SAME namespace, reach the whole backend tier by Service name:
kubectl exec -it frontend-pod-1 -- curl http://back-end:80
# — equivalently, by ClusterIP directly (works, but far less readable/portable) —
kubectl exec -it frontend-pod-1 -- curl http://10.107.34.121:80
```

> ⚠️ **Exam Tip:** if `kubectl describe service` shows `Endpoints: <none>`, the fix is
> almost always to correct the Service's `selector` (or the Pods' `labels`) — it is
> never a kube-proxy problem at this stage.

### Key Takeaways / Traps

1. `ClusterIP` is the **default** Service type — a Service with no `type` field at
   all is a ClusterIP Service; this is exactly the type of the built-in `kubernetes`
   Service that always exists in `default` (`kubectl get services` → check its
   `TYPE` column to confirm).
2. ClusterIP Services are **only** reachable from inside the cluster — there is no
   `EXTERNAL-IP` and no node-level port; this is the right choice for
   tier-to-tier traffic (frontend→backend, app→database) that should never be
   exposed publicly.
3. The virtual IP a ClusterIP Service gets is **stable for the Service's lifetime**,
   even as the Pods behind it are replaced — this decoupling is the entire point.
4. `kubectl describe service <name>` shows exactly what to check when things look
   broken: `TargetPort` (does it match the container's actual listening port?),
   `Endpoints` (are there any, and do they change as Pods are replaced?), and
   `Selector` (does it match the backend Pods' `labels`?).
5. Practice-test callouts worth remembering as fast one-liners:
   `kubectl get services` (count/list services and check the default `TYPE` in one
   shot), `kubectl describe service | grep TargetPort` (find `targetPort` without
   reading full YAML), `kubectl get service --show-labels` (count labels on a
   Service directly from the table), and `kubectl describe service` → look for the
   `Endpoints:` line to count attached backend Pods.
6. When a task asks you to expose an *existing* Pod/Deployment rather than write a
   Service from scratch, prefer generating it (`kubectl expose ...`, see topic 23)
   or filling in a provided `service-definition-1.yaml` template and
   `kubectl create -f`-ing it, rather than hand-authoring one from memory.

---

<a id="t22"></a>
## 22. Practice Test - Services

> 🎯 **In one sentence:** a hands-on drill of everything in topics 20-21 — nothing
> new, just practice.

Every lesson this test exercises is already folded into topics 20 and 21's **Key
Takeaways** above. The condensed drill list:

| Command | What it drills |
|---|---|
| `kubectl get services` | Count/list Services and check the default `TYPE` in one shot |
| `kubectl describe service \| grep TargetPort` | Find `targetPort` without reading full YAML |
| `kubectl get service --show-labels` | Count labels on a Service directly from the table |
| `kubectl describe service` → `Endpoints:` | Count attached backend Pods, confirm selector matching worked |
| Filling in a provided `service-definition-1.yaml` and `kubectl create -f` | Practicing the "template → fill in → create" workflow instead of hand-authoring from memory |

---

<a id="t23"></a>
## 23. Imperative Commands with kubectl

> 🎯 **In one sentence:** never hand-write YAML under time pressure — generate it with
> `--dry-run=client -o yaml`, edit only what's missing, then create it for real.

**Concept (in plain terms):** Kubernetes objects can be managed three different ways,
and knowing **when to use which** is a core exam-speed skill, not just a style
preference:

| Approach | How | Trade-off |
|---|---|---|
| **Imperative commands** | `kubectl run`, `kubectl create deployment`, `kubectl expose`, `kubectl scale`, `kubectl set image`, `kubectl edit`, ... | Fastest to type; no file to manage; but not repeatable/version-controlled, and not every field is settable via flags. |
| **Imperative object configuration** | Hand/generated YAML + `kubectl create -f` / `kubectl replace -f` / `kubectl delete -f` | Full control over every field; but you must specify the *entire* object every time, and `create`/`replace` fail or clobber if state doesn't match exactly what you expect. |
| **Declarative object configuration** | Hand/generated YAML + `kubectl apply -f` (file or whole directory) | Kubernetes computes and merges just the diff against live state; safest for ongoing management, but slower to reason about "what will actually change". |

**The single most important exam-speed trick in this whole course:** never
hand-write YAML from scratch under time pressure. Instead, use an imperative command
with **`--dry-run=client -o yaml`** to have `kubectl` generate a correct, valid YAML
skeleton for you — for *any* object type it supports — then redirect it to a file,
edit only the field(s) the task actually needs (fields no imperative flag can set:
`resources`, `probes`, `volumes`, extra `env`, etc.), and create it for real.

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
#   --dry-run=client   -> validate and print the object, but do NOT send it to the
#                          apiserver / do NOT actually create anything
#   -o yaml            -> print it as YAML instead of the default human table

vi pod.yaml                 # add whatever --run can't set (probes, resources, ...)
kubectl create -f pod.yaml  # now actually create it
```

### Quick reference — generating each object type imperatively

```bash
# PODS
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --port=8080                  # container port
kubectl run nginx --image=nginx --labels=app=nginx,tier=web   # -l is the short form
kubectl run nginx --image=nginx --env=ENVIRONMENT=prod
kubectl run nginx --image=nginx --dry-run=client -o yaml
# override the container's command/args:
kubectl run busybox --image=busybox --command -- sleep 3600
# quick disposable pod for testing, auto-deleted when it exits:
kubectl run tmp --image=busybox --rm -it --restart=Never -- date

# DEPLOYMENTS
kubectl create deployment nginx --image=nginx
kubectl create deployment nginx --image=nginx --replicas=4
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
# NOTE: --replicas isn't available on every kubectl version's `create deployment` —
# if it's missing/ignored, just scale afterward:
kubectl scale deployment nginx --replicas=4

# SERVICES
kubectl expose pod nginx --port=80 --target-port=8080 --name=nginx-service
kubectl expose deployment nginx --port=80 --target-port=8080
kubectl create service clusterip my-svc --tcp=80:8080
kubectl create service nodeport my-svc --tcp=80:8080 --node-port=30080
# CAUTION: `kubectl create service ...` sets spec.selector to app=<name> by default —
# this will NOT necessarily match your real Pods' labels; always check/fix the
# generated selector before relying on it (kubectl expose instead copies the
# selector from the Pod/Deployment you're exposing, which is usually safer).

# JOBS / CRONJOBS
kubectl create job my-job --image=busybox -- date
kubectl create cronjob my-cronjob --image=busybox --schedule="*/1 * * * *" -- date

# CONFIGMAPS
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2
kubectl create configmap my-config --from-file=path/to/config.properties

# SECRETS
kubectl create secret generic my-secret --from-literal=password=mypass
kubectl create secret generic my-secret --from-file=path/to/secret.txt

# NAMESPACES
kubectl create namespace dev-ns      # or: kubectl create ns dev-ns
```

### Diagram — The exam-speed pipeline: generate → edit → create

```
  kubectl run/create <type> ... --dry-run=client -o yaml   ──►  STDOUT (valid YAML,
       (no flag exists for every field you might need:              nothing created
        resources, probes, volumes, extra selector rules...)        on the cluster yet)
                                                                        │
                                                              redirect: > file.yaml
                                                                        │
                                                                        ▼
                                                              vi file.yaml
                                                              (hand-add ONLY the
                                                               missing field(s))
                                                                        │
                                                                        ▼
                                             kubectl create -f file.yaml   (or apply -f)
                                                                        │
                                                                        ▼
                                                        object now actually exists,
                                                        built mostly by kubectl itself —
                                                        far less typing, far fewer typos
                                                        than authoring YAML from scratch
```

### Worked Example — Pod + same-named ClusterIP Service in a single command

```bash
# Task: "create a pod called httpd using httpd:alpine, then a ClusterIP service by
# the same name, target port 80" — doable in ONE imperative command via --expose:
kubectl run httpd --image=httpd:alpine --expose --port=80

kubectl get pods
# httpd   1/1   Running

kubectl get services
# httpd   ClusterIP   10.108.12.4   <none>   80/TCP
#         ^ same name as the pod, selector auto-set to match the pod's own labels
```

### Worked Example — Namespaced object creation, and combining flags

```bash
# Deploy straight into a specific namespace, no YAML, replicas set inline
kubectl create deployment redis-deploy -n dev-ns --image=redis --replicas=2

# Create a pod with a label attached in the same breath (-l is short for --labels)
kubectl run redis --image=redis:alpine -l tier=db

# Expose an existing pod on a specific service port (independent of the pod's own
# container port) and name the resulting Service explicitly
kubectl expose pod redis --port=6379 --name=redis-service
```

### Worked Example — Speed aliases used constantly under exam time pressure

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"     # k run nginx --image=nginx $do > pod.yaml
export now="--force --grace-period=0"    # k delete pod x $now  (skip graceful shutdown)

# Forgot exact field names/structure for a resource? Ask kubectl instead of guessing:
kubectl explain deployment.spec.strategy
kubectl api-resources                     # find the right kind/shortname fast
```

> ⚠️ **Exam Tip:** `kubectl create service ...` sets `selector` to `app=<name>` by
> default, which may not match your real Pods' labels at all — `kubectl expose`
> instead copies the selector straight from the Pod/Deployment you're exposing,
> which is almost always the safer choice.

### Key Takeaways / Traps

1. **`--dry-run=client -o yaml`** is the single highest-leverage exam trick in this
   entire course — it works for *any* object `kubectl run`/`kubectl create ...` can
   generate (Pods, Deployments, Jobs, CronJobs, Services, ConfigMaps, Secrets), and
   turns "write correct YAML from memory" into "generate it, then edit only what's
   missing."
2. `kubectl run` **only ever creates a Pod** — since the `--generator` flag was
   removed (Kubernetes 1.18+ deprecated it, gone by 1.19+), there is no imperative
   way to have `kubectl run` create a Deployment or ReplicaSet directly; use
   `kubectl create deployment` for that instead.
3. `kubectl create deployment <name> --image=<image> --replicas=<n>` is the fast
   path for Deployments — but the `--replicas` flag isn't guaranteed on every
   version; if it's not respected, fall back to `kubectl scale deployment <name>
   --replicas=<n>` immediately afterward.
4. `kubectl expose <pod|deployment|rc> ...` copies the **target object's own
   labels** into the generated Service's `selector` automatically — safer than
   `kubectl create service ...`, which defaults `selector` to `app=<given-name>`
   and can silently produce a Service with **zero** matching Endpoints if that
   doesn't happen to match your real Pods' labels. Always verify with
   `kubectl describe service <name>` → `Endpoints:` after using `create service`.
5. `kubectl run <name> --image=<image> --expose --port=<port>` creates **both** a
   Pod and a same-named `ClusterIP` Service targeting that port, in one command —
   the fastest pattern for "create X and expose it" tasks.
6. `-l`/`--labels` on `kubectl run` (e.g. `-l tier=db`) and `-n`/`--namespace` on any
   `create`/`run` command let you set labels/namespace inline without a second
   command or edit step.
7. Know the **imperative vs. declarative** distinction cold: `kubectl create -f`
   fails if the object already exists; `kubectl replace -f` requires the object to
   already exist and overwrites it wholesale; `kubectl apply -f` creates it if
   missing and merges/patches just the diff if it exists — `apply` is what you'd
   use for ongoing, repeatable management, but `create`/`run` + `--dry-run` is what
   you reach for to move fast on a one-off exam task.
8. `kubectl create configmap`/`kubectl create secret generic` both support
   `--from-literal=key=value` (repeatable) and `--from-file=path` (uses the
   filename as the key, file contents as the value) — no YAML required for either.
9. When an imperative flag simply doesn't exist for the field you need (resource
   `limits`/`requests`, liveness/readiness `probes`, extra `volumes`,
   `env.valueFrom`, etc.), that's the exact trigger to generate the base object with
   `--dry-run=client -o yaml`, then hand-edit just that one field before creating it
   for real — don't try to author the whole object by hand.
10. `kubectl create job <name> --image=<image> -- <cmd>` and
    `kubectl create cronjob <name> --image=<image> --schedule="<cron>" -- <cmd>` are
    the imperative generators for Jobs/CronJobs — the `-- <cmd>` suffix overrides
    the container's default command, exactly like `kubectl run ... --command --
    <cmd>` does for Pods.

---

<a id="t24"></a>
## 24. Practice Test - Imperative Commands

> 🎯 **In one sentence:** a hands-on drill of everything in topic 23 — nothing new,
> just practice.

Every lesson this test exercises is already folded into topic 23's **Key Takeaways**
above. The condensed drill list:

| Command | What it drills |
|---|---|
| `kubectl run <name> --image=<image> --expose --port=<port>` | Create a Pod + same-named ClusterIP Service in one shot |
| `kubectl run <name> --image=<image> --dry-run=client -o yaml > file.yaml` | Generate-then-edit-then-create workflow for a Pod |
| `kubectl create deployment <name> --image=<image> --replicas=<n>` | Fast Deployment creation, falling back to `kubectl scale` if `--replicas` isn't honored |
| `kubectl expose deployment <name> --port= --target-port=` | Exposing an existing Deployment with a correctly-copied selector |
| `kubectl create configmap` / `kubectl create secret generic` with `--from-literal=` | No-YAML ConfigMap/Secret creation |
| `kubectl create job` / `kubectl create cronjob --schedule=` | Imperative Job/CronJob generation |

---

<a id="t25"></a>
## 25. Attachments

> 🎯 **In one sentence:** just two links to the course's presentation decks — no
> technical content of its own.

This lecture has no Kubernetes concepts, YAML, or commands — it's purely a pointer to
the slide decks used throughout this section, for anyone who wants the original
presentation visuals to go with these notes:

- [Presentation Deck - 1](https://kodekloud.com/topic/attachments/)
- [Presentation Deck - 2](https://kodekloud.com/topic/download-presentation-deck-for-this-section-1/)

---

*Notes complete for the full Core Concepts module, topics 01–25 (Section
Introduction through Attachments) — see the [Table of Contents](#toc) or
[Cheat Sheet](#cheat-sheet) above to jump to any topic.*
