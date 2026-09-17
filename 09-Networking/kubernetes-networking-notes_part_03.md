# Networking — Complete Notes (Part 3 of 4)

> **Part 3 of 4** — covers Sections 15–21 (IPAM Weave, Practice Test — Networking Weave,
> Service Networking, Practice Test — Service Networking, DNS in Kubernetes, CoreDNS in
> Kubernetes, Practice Test — CoreDNS in Kubernetes).
> Previous: [kubernetes-networking-notes_part_02.md](kubernetes-networking-notes_part_02.md)
> (Sections 08–14).
> Next: [kubernetes-networking-notes_part_04.md](kubernetes-networking-notes_part_04.md)
> (Sections 22–26 + Quick Revision Checklist).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/09-Networking/`

---

## 15. IPAM Weave

- Video reference: *IPAM weave* — https://kodekloud.com/topic/ipam-weave/
- Topic: **IP Address Management (IPAM)** in the Kubernetes Cluster — specifically how the Weave Net CNI plugin assigns IP addresses to Pods across the cluster.

### IP Address Management in the Kubernetes Cluster

- **Inferred context (image `net3.PNG`):** this image is very likely a generic diagram illustrating the *problem* IPAM solves in Kubernetes — every Pod in the cluster must get a **unique IP address**, with no two Pods (even on different nodes) ever colliding. It most plausibly shows multiple nodes, each running a set of Pods, and calls out that some component (the CNI plugin — here, Weave) is responsible for carving up a single cluster-wide Pod-network CIDR block and handing out non-overlapping chunks/addresses to each node so that Pod IPs never conflict cluster-wide. This sets up the motivating question answered by the next image: *how exactly does Weave do this allocation?*

### How weaveworks Manages IP addresses in the Kubernetes Cluster

- **Inferred context (image `net4.PNG`):** this image is very likely a diagram of Weave Net's specific IPAM scheme, showing:
  - Weave Net's **default overall Pod IP address range/subnet is `10.32.0.0/12`** (giving usable addresses from `10.32.0.0` through `10.47.255.255`) — this is Weave's out-of-the-box `IPALLOC_RANGE`, distinct from the Service network range (`--service-cluster-ip-range`, e.g. `10.96.0.0/12`, covered in file 17) and distinct from whatever CIDR another CNI plugin like Flannel might default to.
  - Unlike some CNI plugins that statically pre-carve a fixed `/24` per node, **Weave uses a decentralized, dynamic IPAM system**: each Weave peer (the `weave` agent running on every node, seen again in file 16) participates in a distributed, consensus-based allocation scheme (gossip-based) where peers claim/reserve chunks of the overall `10.32.0.0/12` range from each other on demand as Pods are scheduled, rather than being handed one fixed subnet up front.
  - This is consistent with what's observed hands-on in file 16's practice test: the Weave bridge/interface's IP on a node begins with `10.` — matching the `10.32.0.0/12` default range described here.
  - **Exam-relevant:** know that Weave Net's **default Pod CIDR is `10.32.0.0/12`**, that it is allocated **per-cluster** (not a fixed static per-node block), and that this is configurable at install time (e.g. via the `IPALLOC_RANGE` environment variable on the Weave DaemonSet) if it would otherwise clash with existing network ranges in an environment.

### References Docs

- https://www.weave.works/docs/net/latest/kubernetes/kube-addon/
- https://kubernetes.io/docs/concepts/cluster-administration/networking/

---

## 16. Practice Test — Networking Weave

- Practice test reference: https://kodekloud.com/topic/practice-test-networking-weave/
- Full step-by-step lab-guide walkthrough of the solutions:

1. **"How many Nodes are part of this cluster?"**
   ```bash
   kunbectl get nodes
   ```
   *(Note: the source text has a typo, `kunbectl` instead of `kubectl` — the correct command is `kubectl get nodes`.)*
   - **Answer: `2`**

2. **"What is the Networking Solution used by this cluster?"**
   - Two ways to check this:
     ```bash
     kubectl get pods -n kube-system
     ```
     ```bash
     ls -l /opt/cni/bin
     ```
   - In both you see evidence of:
   - **Answer: `weave`**
   - (The `kube-system` Pod list shows `weave-net-...` Pods; `/opt/cni/bin` shows the Weave CNI binary installed there.)

3. **"How many weave agents/peers are deployed in this cluster?"**
   ```bash
   kubectl get pods -n kube-system
   ```
   - **Answer: `2`**

4. **"On which nodes are the weave peers present?"**
   ```bash
   kubectl get pods -n kube-system -o wide
   ```
   - **Answer: `One on every node`**
   - (Consistent with Weave's peer agent being deployed as a DaemonSet — one Pod per node — the same DaemonSet pattern later confirmed for `kube-proxy` in file 18.)

5. **"Identify the name of the bridge network/interface created by weave on each node."**
   - At either host:
     ```bash
     ip addr list
     ```
   - **Answer: `weave`**
   - In actual fact, the network interface is `weave` and the bridge is implemented by `vethwe-datapath@vethwe-bridge` and `vethwe-bridge@vethwe-datapath`.

6. **"What is the POD IP address range configured by weave?"**
   - Examine the output of the previous command for the `weave` interface. Note its IP begins with `10.`, so:
   - **Answer: `10.X.X.X`**
   - (Ties directly back to file 15's default Weave range of `10.32.0.0/12`.)

7. **"What is the default gateway configured on the PODs scheduled on node01?"**
   - Deduce this from the answer to the previous question. Since we know weave's IP range, its gateway must be on the same network. However, we can verify that by starting a Pod which is known to contain the `ip` tool.
   - Remember this container image: https://github.com/wbitt/Network-MultiTool — extremely useful for debugging cluster networking issues.
     ```bash
     kubectl run testpod --image=wbitt/network-multitool
     ```
   - Wait for it to be running.
     ```bash
     kubectl exec -it testpod -- ip route
     ```
   - Note the first line of the output — this is the answer (the default route / gateway line, showing the gateway IP address that lives on the same `10.x.x.x` Weave-managed subnet).

**Exam-relevant takeaway:** identifying a cluster's CNI plugin is a two-command habit — `kubectl get pods -n kube-system` (look for plugin-named Pods, e.g. `weave-net-...`) and/or `ls -l /opt/cni/bin` (look for the plugin's binary). Weave deploys **one peer/agent Pod per node** (a DaemonSet), creates a bridge/interface literally named `weave` on every node, and its Pod CIDR (and therefore each Pod's default gateway) is derived from Weave's `10.32.0.0/12` default range. When you need to actually poke at cluster networking from inside a Pod (`ip route`, `ip addr`, `curl`, `ping`, `dig`, `nslookup`, etc.), spin up a debugging Pod using an image built for it, such as `wbitt/network-multitool`.

---

## 17. Service Networking

- Video reference: *Service Networking* — https://kodekloud.com/topic/service-networking/
- Topic: how Kubernetes **Services** get their IP addresses, how they're wired up to backing Pods, and how to inspect the underlying `kube-proxy`/iptables machinery that makes Service networking actually work.

### Service Types

- **ClusterIP** — the default Service type, exposing the Service only on an internal cluster IP, reachable from within the cluster:
  ```yaml
  # clusterIP.yaml

  apiVersion: v1
  kind: Service
  metadata:
    name: local-cluster
  spec:
    ports:
    - port: 80
      targetPort: 80
    selector:
      app: nginx
  ```
- **NodePort** — exposes the Service (in addition to getting a ClusterIP) on a static port on every node's IP, making it reachable from outside the cluster:
  ```yaml
  # nodeportIP.yaml

  apiVersion: v1
  kind: Service
  metadata:
    name: nodeport-wide
  spec:
    type: NodePort
    ports:
    - port: 80
      targetPort: 80
    selector:
      app: nginx
  ```

### To create the service

```
$ kubectl create -f clusterIP.yaml
service/local-cluster created

$ kubectl create -f nodeportIP.yaml
service/nodeport-wide created
```

### To get the Additional Information

```
$ kubectl get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          1m   10.244.1.3   node01   <none>           <no
```
- Notes the backing Pod's actual **Pod IP** (`10.244.1.3`) and the **node** it's scheduled on (`node01`) — this is the endpoint address the Service will ultimately forward traffic to, matched via the Service's `selector: app: nginx`.

### To get the Service

```
$ kubectl get service
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        5m22s
local-cluster   ClusterIP   10.101.67.139   <none>        80/TCP         3m
nodeport-wide   NodePort    10.102.29.204   <none>        80:30016/TCP   2m
```
- **How a Service gets its IP:** every Service (regardless of type — `ClusterIP` or `NodePort`) is automatically allocated a virtual **`CLUSTER-IP`** from the cluster's configured Service IP range at creation time — the kube-apiserver hands out the next free address out of that range (e.g. `10.101.67.139` for `local-cluster`, `10.102.29.204` for `nodeport-wide`). Note the built-in `kubernetes` Service itself (`10.96.0.1`) — the Service through which Pods reach the API server — also comes from this same range and is always the first IP allocated in it.
- For `nodeport-wide`, the `PORT(S)` column shows **`80:30016/TCP`** — `80` is the Service's own `port` (its ClusterIP-facing port), and `30016` is the **NodePort** automatically allocated (from the default NodePort range, `30000–32767`) on every node's IP, making the Service additionally reachable at `<any-node-IP>:30016`.

### To check the Service Cluster IP Range

```
$ ps -aux | grep kube-apiserver
--secure-port=6443 --service-account-key-file=/etc/kubernetes/pki/sa.pub --
service-cluster-ip-range=10.96.0.0/12
```
- The **`--service-cluster-ip-range`** flag on the `kube-apiserver` process defines the entire pool of addresses (`10.96.0.0/12` here) that all Service `CLUSTER-IP`s are allocated from — this is exactly why `kubernetes`, `local-cluster`, and `nodeport-wide` above all got IPs inside `10.96.0.0/12` (`10.96.0.1`, `10.101.67.139`, `10.102.29.204`).

### To check the rules created by kube-proxy in the iptables

```
$ iptables -L -t nat | grep local-cluster
KUBE-MARK-MASQ  all  --  10.244.1.3           anywhere             /* default/local-cluster: */
DNAT       tcp  --  anywhere             anywhere             /* default/local-cluster: */ tcp to:10.244.1.3:80
KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.101.67.139        /* default/local-cluster: cluster IP */ tcp dpt:http
KUBE-SVC-SDGXHD6P3SINP7QJ  tcp  --  anywhere             10.101.67.139        /* default/local-cluster: cluster IP */ tcp dpt:http
KUBE-SEP-GEKJR4UBUI5ONAYW  all  --  anywhere             anywhere             /* default/local-cluster: */
```
- This is the concrete evidence of **how `kube-proxy` load-balances traffic to a Service's backing Pod endpoints**: for every Service, `kube-proxy` (running on every node, in `iptables` mode here) programs a chain of NAT rules in the kernel's `nat` table:
  - A **`KUBE-SVC-<hash>`** chain is the entry point matched by traffic destined for the Service's `CLUSTER-IP:port` (here `10.101.67.139:80`, tagged `cluster IP`) — this is the rule that represents the Service object itself.
  - From the `KUBE-SVC-<hash>` chain, traffic is sent on to one of one-or-more **`KUBE-SEP-<hash>`** ("Service EndPoint") chains — one such chain exists **per backing Pod endpoint**. When a Service has multiple healthy endpoints, `kube-proxy` installs multiple `KUBE-SEP-*` rules under the `KUBE-SVC-*` chain, each selected with an equal-probability `statistic --mode random` rule — this is the actual mechanism by which Service traffic gets **load-balanced across Pod endpoints** in iptables mode (each new connection has an equal chance of landing on any one endpoint). Here there is only one backing Pod (`10.244.1.3`), so there's just one `KUBE-SEP-GEKJR4UBUI5ONAYW` chain.
  - The matched **`KUBE-SEP-*`** chain performs the actual **DNAT** (Destination NAT), rewriting the packet's destination from the Service's ClusterIP to the real backing Pod's IP:port — seen here as `DNAT tcp ... to:10.244.1.3:80`.
  - The **`KUBE-MARK-MASQ`** chain marks a packet so that it will later be **SNAT'd/masqueraded** (source address rewritten) by a later rule in the `POSTROUTING` chain — this happens for traffic originating from **outside** the Pod network (`!10.244.0.0/16`) destined for the Service's cluster IP, and also for traffic sourced *from* the Pod itself (`10.244.1.3`) — ensuring return traffic routes back correctly regardless of whether the client was inside or outside the Pod CIDR.
  - **Exam-relevant:** this rule chain (`KUBE-SERVICES` → `KUBE-SVC-<hash>` → `KUBE-SEP-<hash>` → DNAT to Pod IP, with `KUBE-MARK-MASQ` handling the SNAT/masquerade side) is the classic iptables-mode kube-proxy implementation of a Service. (kube-proxy can alternatively run in **IPVS mode**, using the kernel's IP Virtual Server to do the same load-balancing job more efficiently via IPVS scheduling algorithms rather than long iptables rule chains — general context supplementing what this lesson's `iptables`-only walkthrough shows.)

### To check the logs of kube-proxy

- This file location may vary depending on your installation process.
  ```
  $ cat /var/log/kube-proxy.log
  ```

#### References Docs

- https://kubernetes.io/docs/concepts/services-networking/service/

---

## 18. Practice Test — Service Networking

- Practice test reference: https://kodekloud.com/topic/practice-test-service-networking/
- Full step-by-step lab-guide walkthrough of the solutions:

1. **"What network range are the nodes in the cluster part of?"**
   ```
   kubectl get nodes -o wide
   ```
   - Note the `INTERNAL-IP` column to derive:
   - **Answer: `192.20.116.0/24`**

2. **"What is the range of IP addresses configured for PODs on this cluster?"**
   ```
   kubectl get pods -A -o wide
   ```
   - From this list, **exclude the static control-plane Pods** like `kube-apiserver`, as these run on the **host network**, not the Pod network. From the remaining Pods we can derive:
   - **Answer: `10.244.0.0/16`**

3. **"What is the IP Range configured for the services within the cluster?"**
   ```
   kubectl get service -A
   ```
   - Note the `CLUSTER-IP` column to derive:
   - **Answer: `10.96.0.0/12`**

4. **"How many kube-proxy pods are deployed in this cluster?"**
   ```
   kubectl get pod -n kube-system | grep kube-proxy
   ```
   - Count the results.

5. **"What type of proxy is the kube-proxy configured to use?"**
   - From the output of the above question, you have two kube-proxy pods, e.g.:
     ```
     controlplane ~ kubectl get pod -n kube-system | grep kube-proxy
     kube-proxy-rtr8p                       1/1     Running   0             56m
     kube-proxy-t7w8f                       1/1     Running   0             56m
     ```
   - Pick either and check its logs — the answer is there:
     ```
     k logs -n kube-system kube-proxy-rtr8p
     ```
   - (The kube-proxy startup log lines explicitly report which proxier mode — e.g. `iptables` or `ipvs` — it initialized with.)

6. **"How does this Kubernetes cluster ensure that a kube-proxy pod runs on all nodes in the cluster?"**
   ```
   kubectl get all -n kube-system
   ```
   - From this, you can see that `kube-proxy` is a **`daemonset`**.

**Exam-relevant takeaway:** three separate CIDR ranges must never be confused — the **node network** (from `kubectl get nodes -o wide`'s `INTERNAL-IP`), the **Pod network** (from `kubectl get pods -A -o wide`, excluding host-network static control-plane Pods), and the **Service network** (from `kubectl get service -A`'s `CLUSTER-IP`, matching the `--service-cluster-ip-range` flag seen in file 17). `kube-proxy` is deployed as a **DaemonSet** (one Pod per node) precisely so that Service-routing rules exist locally on every node; its logs (`kubectl logs -n kube-system <kube-proxy-pod>`) directly reveal which proxy mode (`iptables` or `ipvs`) it's running in.

---

## 19. DNS in Kubernetes

- Video reference: *DNS in Kubernetes* — https://kodekloud.com/topic/dns-in-kubernetes/
- Topic: how DNS resolution works **inside** a Kubernetes cluster — the naming schemes used to reach both individual **Pods** and **Services** by name rather than by IP.

### Pod DNS Record

- The DNS resolution scheme for a Pod:
  ```
  <POD-IP-ADDRESS>.<namespace-name>.pod.cluster.local
  ```
  - The Pod's IP address has its dots replaced with dashes in the DNS name.
  > Example — Pod is located in a default namespace:
  ```
  10-244-1-10.default.pod.cluster.local
  ```
- Worked example:
  ```
  # To create a namespace
  $ kubectl create ns apps

  # To create a Pod
  $ kubectl run nginx --image=nginx --namespace apps

  # To get the additional information of the Pod in the namespace "apps"
  $ kubectl get po -n apps -owide
  NAME    READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
  nginx   1/1     Running   0          99s   10.244.1.3   node01   <none>           <none>

  # To get the dns record of the nginx Pod from the default namespace
  $ kubectl run -it test --image=busybox:1.28 --rm --restart=Never -- nslookup 10-244-1-3.apps.pod.cluster.local
  Server:    10.96.0.10
  Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

  Name:      10-244-1-3.apps.pod.cluster.local
  Address 1: 10.244.1.3
  pod "test" deleted

  # Accessing with curl command
  $ kubectl run -it nginx-test --image=nginx --rm --restart=Never -- curl -Is http://10-244-1-3.apps.pod.cluster.local
  HTTP/1.1 200 OK
  Server: nginx/1.19.2
  ```
  - Note the DNS server that answered the query: **`10.96.0.10`**, identified back as `kube-dns.kube-system.svc.cluster.local` — this is the cluster's internal DNS Service (backed by CoreDNS, covered in full in file 20).
  - The Pod's DNS name (`10-244-1-3.apps.pod.cluster.local`) correctly resolves back to its actual Pod IP (`10.244.1.3`), and is directly usable as an HTTP hostname (`curl http://10-244-1-3.apps.pod.cluster.local` succeeds, `200 OK`).
  - **Exam-relevant:** Pod DNS records under the `pod.cluster.local` zone are **not created/enabled by default** in all cluster setups — this per-Pod A-record scheme depends on the DNS addon; the far more commonly used and always-relied-upon scheme is the **Service** DNS record, covered next.

### Service DNS Record

- The DNS resolution scheme for a Service:
  ```
  <service-name>.<namespace-name>.svc.cluster.local
  ```
  > Example — Service is located in a default namespace:
  ```
  web-service.default.svc.cluster.local
  ```
- Worked example — Pod and Service located in the `apps` namespace:
  ```
  # Expose the nginx Pod
  $ kubectl expose pod nginx --name=nginx-service --port 80 --namespace apps
  service/nginx-service exposed

  # Get the nginx-service in the namespace "apps"
  $ kubectl get svc -n apps
  NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
  nginx-service   ClusterIP   10.96.120.174   <none>        80/TCP    6s

  # To get the dns record of the nginx-service from the default namespace
  $ kubectl run -it test --image=busybox:1.28 --rm --restart=Never -- nslookup nginx-service.apps.svc.cluster.local
  Server:    10.96.0.10
  Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

  Name:      nginx-service.apps.svc.cluster.local
  Address 1: 10.96.120.174 nginx-service.apps.svc.cluster.local
  pod "test" deleted

  # Accessing with curl command
  $ kubectl run -it nginx-test --image=nginx --rm --restart=Never -- curl -Is http://nginx-service.apps.svc.cluster.local
  HTTP/1.1 200 OK
  Server: nginx/1.19.2
  ```
  - The query is made **from the `default` namespace** (via a throwaway `test` Pod created with no explicit `--namespace`), successfully resolving a Service that lives in a **different** namespace (`apps`) — this only works because the query used the **fully-qualified** name (`nginx-service.apps.svc.cluster.local`), including the target namespace explicitly. (A bare `nginx-service` lookup from the `default` namespace would *not* resolve, since the Pod's own default DNS search path only auto-appends `default.svc.cluster.local`, not `apps.svc.cluster.local` — this exact point is explored further via CoreDNS's `/etc/resolv.conf` `search` line in file 20.)
  - The Service's DNS name resolves to its **`CLUSTER-IP`** (`10.96.120.174`), not to any individual backing Pod's IP — consistent with a Service being a stable virtual IP load-balanced (via kube-proxy, per file 17) across whichever Pods currently match its selector.

#### References Docs

- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/

---

## 20. CoreDNS in Kubernetes

- Video reference: *CoreDNS in Kubernetes* — https://kodekloud.com/topic/coredns-in-kubernetes/
- Topic: **CoreDNS** is the DNS server implementation that actually provides the `pod.cluster.local` / `svc.cluster.local` name resolution introduced in file 19 — this lesson looks at how CoreDNS itself is deployed, configured, and how to verify/debug DNS resolution end-to-end.

### To view the Pod

```
$ kubectl get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
coredns-66bff467f8-2vghh                  1/1     Running   0          53m
coredns-66bff467f8-t5nzm                  1/1     Running   0          53m
```
- CoreDNS runs as a set of Pods (here, **2 replicas**) inside the `kube-system` namespace.

### To view the Deployment

```
$ kubectl get deployment -n kube-system
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
coredns                   2/2     2            2           53m
```
- CoreDNS Pods are managed by a **Deployment** named `coredns` (not a DaemonSet — unlike `kube-proxy` in file 18, CoreDNS doesn't need to run on every node; it just needs enough replicas for availability/load, and is reached via its Service ClusterIP from any node).

### To view the configmap of CoreDNS

```
$ kubectl get configmap -n kube-system
NAME                                 DATA   AGE
coredns                              1      52m
```
- CoreDNS's configuration (its **Corefile**) is stored as a **ConfigMap** named `coredns`, and mounted into the CoreDNS Pods as a file.

### CoreDNS Configuration File

```
$ kubectl describe cm coredns -n kube-system

Corefile:
---
.:53 {
    errors
    health {       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
}
```
- This is the **Corefile** — CoreDNS's plugin-chain configuration format. Reading it plugin-by-plugin:
  - **`.:53`** — the server block: CoreDNS listens for **all domains (`.`)** on **port 53** (the standard DNS port).
  - **`errors`** — errors are logged to stdout.
  - **`health { lameduck 5s }`** — exposes a `/health` HTTP endpoint used for Kubernetes health checks (liveness); `lameduck 5s` delays reporting unhealthy for 5s on shutdown, letting in-flight requests finish before the process exits.
  - **`ready`** — exposes a `/ready` HTTP endpoint for readiness checks.
  - **`kubernetes cluster.local in-addr.arpa ip6.arpa { ... }`** — the core **Kubernetes plugin**: this is what actually implements the `svc.cluster.local` / `pod.cluster.local` DNS resolution scheme from file 19, by querying the Kubernetes API for Service/Endpoint/Pod objects. It's configured for the **`cluster.local`** zone plus the reverse-lookup zones `in-addr.arpa` and `ip6.arpa` (for IP-to-name/PTR lookups).
    - **`pods insecure`** — enables the `pod.cluster.local` A-records (per-Pod DNS, as seen in file 19) in a mode that does not verify the Pod actually exists in that exact namespace before answering (a relaxed/insecure mode, as opposed to `pods verified` or `pods disabled`).
    - **`fallthrough in-addr.arpa ip6.arpa`** — if a reverse-lookup query for these zones can't be answered by the Kubernetes plugin, **fall through** to the next plugin in the chain (rather than immediately returning NXDOMAIN) — allowing a later plugin (here, ultimately `forward`) to have a chance at answering it.
    - **`ttl 30`** — sets the TTL of DNS responses served by this plugin to 30 seconds.
  - **`prometheus :9153`** — exposes CoreDNS's own metrics in Prometheus format on port `9153` (matches the `9153/TCP` port seen on the `kube-dns` Service below).
  - **`forward . /etc/resolv.conf`** — for any query **not** handled by the Kubernetes plugin (i.e. anything outside `cluster.local`/reverse zones — ordinary internet/external DNS names), **forward** the query upstream to whatever nameserver(s) are listed in the CoreDNS Pod's own `/etc/resolv.conf` (which normally reflects the underlying **node's** upstream DNS resolver) — this is CoreDNS's **stubDomain/upstream forwarding** mechanism, letting cluster DNS transparently proxy external lookups (e.g. `google.com`) out to the real world while still owning internal cluster names itself.
  - **`cache 30`** — caches DNS responses for 30 seconds, reducing load on both the Kubernetes API and any upstream forwarders.
  - **`loop`** — detects DNS forwarding loops (e.g. accidentally forwarding back to itself) and halts CoreDNS with a fatal error if one is found.
  - **`reload`** — allows CoreDNS to automatically reload this Corefile if the underlying ConfigMap changes, without needing a Pod restart.
- **Exam-relevant:** the Corefile's **`kubernetes` plugin block** is the single most important piece to recognize — it's what makes CoreDNS "Kubernetes-aware" at all (querying the API server for Service/Endpoint/Pod data to answer `*.svc.cluster.local`/`*.pod.cluster.local` queries), and the **`forward . /etc/resolv.conf`** line is the standard way upstream/external DNS resolution is wired in (equivalent in spirit to a "stubDomain"/conditional-forwarder concept from other DNS servers) — if a cluster needs custom upstream forwarding (e.g. for a specific corporate DNS zone), editing this `forward` line (or adding another `kubernetes`/`forward` server block for a specific zone) in the `coredns` ConfigMap is exactly how it's done.

### To view the Service

```
$ kubectl get service -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   62m
```
- The `kube-dns` Service (a `ClusterIP` Service, historically named `kube-dns` even when CoreDNS is the actual backend rather than the older `kube-dns` server, for compatibility) is what fronts the CoreDNS Pods, exposing DNS on **53/UDP** and **53/TCP**, plus metrics on **9153/TCP**. Its ClusterIP, **`10.96.0.10`**, is exactly the DNS server address seen answering every `nslookup` query throughout file 19.

### To view Configuration into the kubelet

```
$ cat /var/lib/kubelet/config.yaml | grep -A2  clusterDNS
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
```
- This is **how every Pod learns to use `10.96.0.10` as its nameserver**: the **kubelet's** own configuration file (`/var/lib/kubelet/config.yaml`) has `clusterDNS: [10.96.0.10]` and `clusterDomain: cluster.local` set — the kubelet uses these values to populate every Pod's `/etc/resolv.conf` at Pod-creation time (see below).

### To view the fully qualified domain name

- With the `host` command, we get the fully qualified domain name (FQDN):
  ```
  $ host web-service
  web-service.default.svc.cluster.local has address 10.106.112.101

  $ host web-service.default
  web-service.default.svc.cluster.local has address 10.106.112.101

  $ host web-service.default.svc
  web-service.default.svc.cluster.local has address 10.106.112.101

  $ host web-service.default.svc.cluster.local
  web-service.default.svc.cluster.local has address 10.106.112.101
  ```
  - All four increasingly-qualified forms resolve to the **same** address — demonstrating the DNS **search domain** mechanism in action: a short name like plain `web-service` gets progressively expanded against each entry in the resolver's `search` list (seen next, in `/etc/resolv.conf`) until a match is found, ultimately arriving at the same full FQDN, `web-service.default.svc.cluster.local`.

### To view the `/etc/resolv.conf` file

```
$ kubectl run -it --rm --restart=Never test-pod --image=busybox -- cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
pod "test-pod" deleted
```
- **`nameserver 10.96.0.10`** — every Pod is configured (by the kubelet, per the `clusterDNS` setting above) to send DNS queries to the CoreDNS-backed `kube-dns` Service ClusterIP.
- **`search default.svc.cluster.local svc.cluster.local cluster.local`** — the ordered list of suffixes appended to a short/unqualified hostname when trying to resolve it — this is exactly the mechanism that let `host web-service` (above) succeed: the resolver tries `web-service.default.svc.cluster.local` first (matching the search list's first entry, and this Pod's own `default` namespace) and finds it.
- **`options ndots:5`** — any name with **fewer than 5 dots** is treated as **not fully qualified**, and the `search` suffixes are tried against it first (in order) before it's ever tried as an absolute name as-is; a name with 5 or more dots is queried as-is (absolute) first. This is a well-known real-world Kubernetes DNS **performance gotcha**: fully-qualifying external lookups (e.g. `www.google.com.`, or lowering `ndots`) avoids unnecessarily trying (and failing) up to 3 extra internal search-suffixed lookups before falling through to the real external name.

### Resolve the Pod

```
$ kubectl get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
test-pod   1/1     Running   0          11m     10.244.1.3   node01   <none>           <none>
nginx      1/1     Running   0          10m     10.244.1.4   node01   <none>           <none>

$ kubectl exec -it test-pod -- nslookup 10-244-1-4.default.pod.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      10-244-1-4.default.pod.cluster.local
Address 1: 10.244.1.4
```
- Confirms the **Pod DNS record** scheme from file 19 end-to-end: the dash-encoded IP address (`10-244-1-4`) plus namespace (`default`) plus `pod.cluster.local` correctly resolves back to the `nginx` Pod's real IP (`10.244.1.4`), with CoreDNS (`10.96.0.10`) answering the query.

### Resolve the Service

```
$ kubectl get service
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP   85m
web-service   ClusterIP   10.106.112.101   <none>        80/TCP    9m

$ kubectl exec -it test-pod -- nslookup web-service.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-service.default.svc.cluster.local
Address 1: 10.106.112.101 web-service.default.svc.cluster.local
```
- Confirms the **Service DNS record** scheme from file 19 end-to-end: `web-service.default.svc.cluster.local` correctly resolves to `web-service`'s `CLUSTER-IP` (`10.106.112.101`).

#### References Docs

- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#services
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pods

---

## 21. Practice Test — CoreDNS in Kubernetes

- Practice test reference: https://kodekloud.com/topic/practice-test-coredns-in-kubernetes/
- **Note on this section's source material:** unlike files 16 and 18, this practice test's source file provides **only the answer key** (each item is a bare "Check the Solution" placeholder with the revealed answer inside a `<details>` block) — the original question text itself is **not present** in the source. Per task instructions, the likely question for each item is reconstructed/inferred below from the answer content and the overall DNS-troubleshooting scenario the later items clearly describe (a `web-service`/`mysql` Service being relocated into a `payroll` namespace, accessed from an `hr` Pod). These reconstructed questions are explicitly flagged as **inferred** — only the commands/answers themselves are verbatim from the source.

1. **Answer: `CoreDNS`**
   - *Inferred question:* "What is the DNS solution/implementation deployed in this cluster?"
   - Likely derived by checking `kubectl get pods -n kube-system` (or `kubectl get deployment -n kube-system`, per file 20) and seeing `coredns-...` Pods.

2. **Answer: `2`**
   - *Inferred question:* "How many replicas of the CoreDNS pod are running in this cluster?"
   - Derived via `kubectl get pods -n kube-system` or `kubectl get deployment coredns -n kube-system` (per file 20's `2/2 READY` output).

3. **Answer: `10.96.0.10`**
   - *Inferred question:* "What is the IP address of the DNS server (the `kube-dns` Service ClusterIP) that Pods use for name resolution?"
   - Derived via `kubectl get service -n kube-system` or `cat /etc/resolv.conf` inside any Pod (per file 20).

4. **Answer:**
   ```
   /etc/coredns/Corefile

   OR

   kubectl -n kube-system describe deployments.apps coredns | grep -A2 Args | grep Corefile
   ```
   - *Inferred question:* "Where is the CoreDNS configuration file (the Corefile) located inside the CoreDNS container, and/or how would you confirm that path from the Deployment spec?"
   - Two valid approaches given: state the well-known mount path directly (`/etc/coredns/Corefile`), or derive it by describing the `coredns` Deployment and grepping its container `Args` for the `-conf` flag pointing at that path.

5. **Answer: `Configured as a ConfigMapObject`**
   - *Inferred question:* "How is the CoreDNS Corefile actually supplied/mounted into the CoreDNS pods — as a baked-in file, or dynamically?"
   - Matches file 20's `kubectl get configmap -n kube-system` / `kubectl describe cm coredns -n kube-system` findings — the Corefile lives in a ConfigMap, mounted as a volume into the Pod at the path found in item 4.

6. **Answer: `CoreDNS`**
   - *Inferred question:* likely a variant/rephrasing of item 1 — e.g. "What is the name of the Deployment/application responsible for DNS resolution in `kube-system`?" (answer given in the general/proper-noun form "CoreDNS").

7. **Answer: `coredns`**
   - *Inferred question:* "What is the exact name of the Deployment object that manages the DNS server pods?" — answer given in the lowercase, literal `kubectl`-object-name form (`coredns`), as directly seen in `kubectl get deployment -n kube-system` in file 20, distinguishing this from item 6's proper-noun answer.

8. **Answer: `cluster.local`**
   - *Inferred question:* "What is the default domain name configured for this Kubernetes cluster?"
   - Derived via `cat /var/lib/kubelet/config.yaml | grep -A2 clusterDNS` (per file 20), reading the `clusterDomain:` field.

9. **Answer: `Ok`**
   - *Inferred question:* likely a validation/checkpoint step in the lab (e.g. "Test that you can resolve a Service by name from within a Pod — does it succeed?") rather than a fact-lookup question — the source's answer is a bare status confirmation (`Ok`) rather than a technical value, consistent with an interactive lab-checker step.

10. **Answer: `web-service`**
    - *Inferred question:* "What is the name of the Service in the `default` namespace that you were asked to resolve/verify DNS for?"

11. **Answer: `web-serivce.default.pod`** *(sic — verbatim from source, including its typo `serivce` for `service`)*
    - *Inferred question:* likely testing recognition of an **invalid/incorrect** DNS name form — note that `.default.pod` (missing `.cluster.local`, and using the Pod-record `.pod` suffix on what is a *Service* name) is **not** a valid resolvable Service DNS name under the schemes taught in files 19–20; this may have been a distractor/incorrect-option answer, or reflects a typo already present in the original course material. Preserved here exactly as given in the source, with the discrepancy flagged rather than silently corrected.

12. **Answer: `web-service.payroll`**
    - *Inferred question:* "The relevant Service (or its backing Deployment/Pod) has now moved to a new namespace called `payroll`. What short-form DNS name would another namespace now use to reach it?" — matches the `<service-name>.<namespace-name>` short form (still resolvable thanks to the `ndots:5`/`search`-suffix mechanism from file 20).

13. **Answer: `web-service.payroll.svc.cluster`**
    - *Inferred question:* a further-qualified variant of item 12's answer — "What is a longer/more-qualified form of the DNS name for reaching the relocated service from another namespace?" (one step short of the fully-qualified `web-service.payroll.svc.cluster.local`).

14. **Full worked solution given in the source (task: update an application to point at a service that has moved to the `payroll` namespace):**
    ```
    kubectl edit deploy webapp

    Search for DB_Host and Change the DB_Host from mysql to mysql.payroll

    spec:
      containers:
      - env:
        - name: DB_Host
          value: mysql.payroll
    ```
    - Edits the `webapp` Deployment's Pod template directly (`kubectl edit deploy webapp`), locating the `DB_Host` environment variable and updating its value from a bare `mysql` (which would only resolve within the `webapp` Pod's own namespace via the `search` suffix mechanism) to the namespace-qualified short form `mysql.payroll` — necessary because the actual `mysql` Service now lives in the separate `payroll` namespace, so the unqualified name would no longer resolve correctly for a Pod outside that namespace.

15. **Full worked solution given in the source (task: verify/record a DNS resolution test):**
    ```
    kubectl exec -it hr -- nslookup mysql.payroll > /root/nslookup.out
    ```
    - Runs `nslookup mysql.payroll` from inside the `hr` Pod, and redirects the output to `/root/nslookup.out` — verifying that the `hr` Pod can now successfully resolve the relocated `mysql` Service using its `<service>.<namespace>` short form, and persisting the proof of that resolution to a file (a common practice-test-lab pattern of "save the command's output to a file so the grader can check it").

**Exam-relevant takeaway:** this practice test walks through the exact same command toolkit as file 20 (`kubectl get pods/deployment/configmap/service -n kube-system`, `kubectl describe cm coredns -n kube-system`, `cat /var/lib/kubelet/config.yaml`, `nslookup`) but applies it to a **namespace-migration troubleshooting scenario**: when a Service moves to a new namespace, every consumer that referenced it by a **bare/short name** (relying on the Pod's own namespace being auto-appended via the `search` list and `ndots:5`) breaks, and must be updated to the **namespace-qualified** form (`<service>.<namespace>`, or fully `<service>.<namespace>.svc.cluster.local`) — exactly the fix demonstrated by editing `webapp`'s `DB_Host` env var from `mysql` to `mysql.payroll`, then confirming with `nslookup` from the consuming Pod.

---

*Continued in [kubernetes-networking-notes_part_04.md](kubernetes-networking-notes_part_04.md)
— Sections 22–26 (Ingress, Ingress Annotations and rewrite-target, Practice Test — CKA
Ingress Networking 1, Practice Test — CKA Ingress Networking 2, Download Presentation
Deck) plus the Quick Revision Checklist for the entire Networking section.*
