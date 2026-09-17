# Networking — Complete Notes (Part 5 of 5)

> **Part 5 of 5** — covers Sections 25–26 (Practice Test — CKA Ingress Networking 2,
> Download Presentation Deck) plus the **Quick Revision Checklist** for the entire
> 09-Networking section (all 26 files).
> Previous: [kubernetes-networking-notes_part_04.md](kubernetes-networking-notes_part_04.md)
> (Sections 22–24).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/09-Networking/`

---

## 25. Practice Test — CKA Ingress Networking 2

- Link: *Practice Test* — https://kodekloud.com/topic/practice-test-cka-ingress-networking-2/
- This test walks through **deploying a full NGINX Ingress Controller from scratch** (namespace, ConfigMap, ServiceAccounts, RBAC, Deployment, Service) and then creating an Ingress resource against it — a much more hands-on/"build it yourself" lab than file 24. Most answers here are actual commands or full manifests, so less inference about the underlying task is required than in file 24.

1. *(Inferred question: an initial explore/connectivity-check step.)*
   - **Answer (verbatim):** `OK`

2. *(Task: "Create a namespace called `ingress-nginx`.")*
   - **Answer (verbatim):**
     ```
     kubectl create namespace ingress-nginx
     ```
   - How to read it: this namespace will host all the Ingress Controller's own objects (ConfigMap, ServiceAccounts, Deployment, Service) — separate from the application namespaces.

3. *(Task: "Create a ConfigMap called `ingress-nginx-controller` in the `ingress-nginx` namespace.")*
   - **Answer (verbatim):**
     ```
     kubectl create configmap ingress-nginx-controller --namespace ingress-nginx
     ```
   - How to read it: this is the same purpose as the `nginx-configuration` ConfigMap in file 22 — a place for NGINX Ingress Controller options — just created empty here and populated by the Deployment's `--configmap=$(POD_NAMESPACE)/ingress-nginx-controller` arg (visible in step 6's YAML).

4. *(Task: "Create the two ServiceAccounts the controller and its admission webhook need.")*
   - **Answer (verbatim):**
     ```
     kubectl create serviceaccount ingress-nginx --namespace ingress-nginx
     kubectl create serviceaccount ingress-nginx-admission --namespace ingress-nginx
     ```
   - How to read it: modern `ingress-nginx` deployments (Helm-chart-based, as evidenced by the `helm.sh/chart` labels in step 6) use **two** ServiceAccounts — one (`ingress-nginx`) for the controller Pod itself (referenced by `serviceAccountName: ingress-nginx` in the Deployment spec), and one (`ingress-nginx-admission`) for a separate validating-webhook job/Pod that validates new Ingress objects before they're admitted.

5. *(Inferred question: "Check whether the correct RBAC (Roles/RoleBindings) already exist for these ServiceAccounts.")*
   - **Answer (verbatim):**
     ```
     Ok

     kubectl get roles,rolebindings --namespace ingress-nginx
     ```
   - How to read it: this is a **read-only inspection** step — the lab environment apparently pre-creates the necessary `Role`/`RoleBinding` (and presumably `ClusterRole`/`ClusterRoleBinding`) objects for the controller's RBAC permissions; the student just confirms they exist rather than creating them from scratch.

6. *(Task: "Fix the issues in `/root/ingress-controller.yaml` — it contains one `Deployment` and one `Service`, each with bugs.")*
   - Inspect/edit the file:
     ```
     vi /root/ingress-controller.yaml
     ```
   - **Four explicit bugs called out in the source, to be fixed via `vi`:**
     1. The **`namespace`** of the Deployment is incorrect.
     2. **Indentation error at line 74** (use `:set nu` in `vi` to turn on line numbers to locate it).
     3. The **`name`** of the Service is incorrect.
     4. **`nodeport`** on the Service is the wrong case — it should be **`nodePort`** (YAML/Kubernetes field names are case-sensitive).
   - **Corrected Deployment (verbatim, full manifest):**
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       labels:
         app.kubernetes.io/component: controller
         app.kubernetes.io/instance: ingress-nginx
         app.kubernetes.io/managed-by: Helm
         app.kubernetes.io/name: ingress-nginx
         app.kubernetes.io/part-of: ingress-nginx
         app.kubernetes.io/version: 1.1.2
         helm.sh/chart: ingress-nginx-4.0.18
       name: ingress-nginx-controller
       namespace: ingress-nginx
     spec:
       minReadySeconds: 0
       revisionHistoryLimit: 10
       selector:
         matchLabels:
           app.kubernetes.io/component: controller
           app.kubernetes.io/instance: ingress-nginx
           app.kubernetes.io/name: ingress-nginx
       template:
         metadata:
           labels:
             app.kubernetes.io/component: controller
             app.kubernetes.io/instance: ingress-nginx
             app.kubernetes.io/name: ingress-nginx
         spec:
           containers:
           - args:
             - /nginx-ingress-controller
             - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
             - --election-id=ingress-controller-leader
             - --watch-ingress-without-class=true
             - --default-backend-service=app-space/default-http-backend
             - --controller-class=k8s.io/ingress-nginx
             - --ingress-class=nginx
             - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
             - --validating-webhook=:8443
             - --validating-webhook-certificate=/usr/local/certificates/cert
             - --validating-webhook-key=/usr/local/certificates/key
             env:
             - name: POD_NAME
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.name
             - name: POD_NAMESPACE
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.namespace
             - name: LD_PRELOAD
               value: /usr/local/lib/libmimalloc.so
             image: registry.k8s.io/ingress-nginx/controller:v1.1.2@sha256:28b11ce69e57843de44e3db6413e98d09de0f6688e33d4bd384002a44f78405c
             imagePullPolicy: IfNotPresent
             lifecycle:
               preStop:
                 exec:
                   command:
                   - /wait-shutdown
             livenessProbe:
               failureThreshold: 5
               httpGet:
                 path: /healthz
                 port: 10254
                 scheme: HTTP
               initialDelaySeconds: 10
               periodSeconds: 10
               successThreshold: 1
               timeoutSeconds: 1
             name: controller
             ports:
             - name: http
               containerPort: 80
               protocol: TCP
             - containerPort: 443
               name: https
               protocol: TCP
             - containerPort: 8443
               name: webhook
               protocol: TCP
             readinessProbe:
               failureThreshold: 3
               httpGet:
                 path: /healthz
                 port: 10254
                 scheme: HTTP
               initialDelaySeconds: 10
               periodSeconds: 10
               successThreshold: 1
               timeoutSeconds: 1
             resources:
               requests:
                 cpu: 100m
                 memory: 90Mi
             securityContext:
               allowPrivilegeEscalation: true
               capabilities:
                 add:
                 - NET_BIND_SERVICE
                 drop:
                 - ALL
               runAsUser: 101
             volumeMounts:
             - mountPath: /usr/local/certificates/
               name: webhook-cert
               readOnly: true
           dnsPolicy: ClusterFirst
           nodeSelector:
             kubernetes.io/os: linux
           serviceAccountName: ingress-nginx
           terminationGracePeriodSeconds: 300
           volumes:
           - name: webhook-cert
             secret:
               secretName: ingress-nginx-admission
     ```
   - **Corrected Service (verbatim, full manifest):**
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       creationTimestamp: null
       labels:
         app.kubernetes.io/component: controller
         app.kubernetes.io/instance: ingress-nginx
         app.kubernetes.io/managed-by: Helm
         app.kubernetes.io/name: ingress-nginx
         app.kubernetes.io/part-of: ingress-nginx
         app.kubernetes.io/version: 1.1.2
         helm.sh/chart: ingress-nginx-4.0.18
       name: ingress-nginx-controller
       namespace: ingress-nginx
     spec:
       ports:
       - port: 80
         protocol: TCP
         targetPort: 80
         nodePort: 30080
       selector:
         app.kubernetes.io/component: controller
         app.kubernetes.io/instance: ingress-nginx
         app.kubernetes.io/name: ingress-nginx
       type: NodePort
     ```
   - Key details worth calling out:
     - Deployment's `namespace: ingress-nginx` and Service's `namespace` (implied same, `ingress-nginx`) must **match** the namespace created in step 2 and referenced by the ServiceAccounts/ConfigMap in steps 3–4.
     - `--default-backend-service=app-space/default-http-backend` — this controller is explicitly configured to route unmatched requests to a default backend Service living in the **`app-space`** namespace (tying this lab back to the same `app-space` app namespace seen in file 24).
     - `--ingress-class=nginx` / `--controller-class=k8s.io/ingress-nginx` — this is how this controller instance identifies itself as the handler for Ingresses using `ingressClassName: nginx` (or the class it's configured for) — directly relevant to the IngressClass concept discussed in file 22's `CLASS` column.
     - `--watch-ingress-without-class=true` — tells this controller to **also** process Ingress objects that don't specify any `ingressClassName`/class annotation at all (relevant since many of the lab's Ingress manifests, e.g. in files 22/24, don't set one).
     - `securityContext.capabilities.add: [NET_BIND_SERVICE]` with `drop: [ALL]` — the controller runs as non-root (`runAsUser: 101`) but is granted the specific `NET_BIND_SERVICE` Linux capability so it can bind to privileged ports (80/443) without needing full root.
     - Service `nodePort: 30080` exposes the controller's port 80 externally on **node port 30080** — this is the actual entry point students hit via the lab's "Ingress" browser button (see step 8).

7. *(Task: "Create an Ingress resource `ingress-wear-watch` in `app-space` routing `/wear` to `wear-service` and `/watch` to `video-service`, both on port 8080, using the current stable API.")*
   - **Answer (verbatim, full manifest — note the source's own indentation is preserved exactly as given, including an apparent inconsistency where `name:`/`port:` under `service:` are not indented one level deeper than `service:` itself):**
     ```yaml
     apiVersion: networking.k8s.io/v1
     kind: Ingress
     metadata:
       name: ingress-wear-watch
       namespace: app-space
       annotations:
         nginx.ingress.kubernetes.io/rewrite-target: /
         nginx.ingress.kubernetes.io/ssl-redirect: "false"
     spec:
       rules:
       - http:
           paths:
           - path: /wear
             pathType: Prefix
             backend:
               service:
               name: wear-service
               port: 
                 number: 8080
           - path: /watch
             pathType: Prefix
             backend:
               service:
               name: video-service
               port:
                 number: 8080
     ```
   - How to read it: this mirrors file 24 step 22's schema (`networking.k8s.io/v1`, `pathType: Prefix`, `backend.service.name` + `backend.service.port.number`), the `rewrite-target: /` + `ssl-redirect: "false"` annotation pair used throughout these labs, and again routes to `video-service` (not `watch-service`) for the `/watch` path — consistent with file 24's finding that the "watch" functionality is actually backed by a Service named `video-service`.

8. *(Task: "Verify the Ingress works by testing it in a browser.")*
   - **Answer (verbatim, instructions):**
     Press the `Ingress` button above the terminal pane. In the browser tab that opens, try appending `/wear` or `/watch` after `labs.kodekloud.com` in the browser address bar.
   - How to read it: this is a manual/interactive validation step — no `kubectl` command — confirming end-to-end that requests to `.../wear` and `.../watch` reach the correct backend application through the newly built Ingress Controller + Ingress resource, exercising the NodePort (`30080`) Service, the controller's routing rules, and the `rewrite-target` path rewriting all together.

- **Exam-relevant takeaway (file 25):** know how to **build an ingress-nginx controller from raw manifests** (namespace → ConfigMap → ServiceAccount(s) → RBAC → Deployment → Service), not just how to use a pre-existing one — the exam may hand you a broken manifest with deliberately introduced YAML bugs (wrong namespace, bad indentation, wrong object name, wrong field-name casing like `nodeport` vs `nodePort`) and expect you to `vi`/`kubectl edit` your way to a working controller. Always double check field-name **casing** (`nodePort`, not `nodeport`) and that every object's `namespace:` is consistent across the whole controller stack.

---

## 26. Download Presentation Deck

- **Full content of this file (verbatim):** a single link — [Download - Presentation Deck](https://kodekloud.com/topic/download-presentation-deck-8/).
- As anticipated: this is a **placeholder page with no inline technical content whatsoever** — no prose, no commands, no YAML — identical in nature/format to the other "Download Presentation Deck" placeholder files seen at the end of other sections in this course (e.g. the Security section's file 30, and the Cluster Maintenance section's file 11). It exists purely to link to a downloadable slide deck accompanying the Networking section's video lectures.

---

## Quick Revision Checklist

*Checklist for the entire 09-Networking section (files 01–26).*

- [ ] **Networking intro / switching, routing, gateways** *(files 01–02)*
  - A **switch** connects devices on the **same** network segment (Layer 2, MAC-address based); create one conceptually with `ip link add <name> type bridge`.
  - A **router**/**gateway** connects **different** networks (Layer 3, IP-based); the **default gateway** is where a host sends traffic destined for networks it has no explicit route for (`ip route add default via <gateway>`).
  - Inspect/manage with `ip link`, `ip addr`, `ip route show`/`route`, `arp`; enable a host to forward packets between interfaces with `echo 1 > /proc/sys/net/ipv4/ip_forward` (live) or `net.ipv4.ip_forward=1` in `/etc/sysctl.conf` + `sysctl --system` (persistent).

- [ ] **DNS prerequisites** *(file 03)*
  - `/etc/hosts` — static local hostname→IP mappings, checked before DNS (per `/etc/nsswitch.conf`'s `hosts: files dns` order).
  - `/etc/resolv.conf` — configures `nameserver` (DNS server IP, can list multiple as fallbacks) and `search` domain(s) used to expand unqualified names.
  - Tools: `nslookup <name>`, `dig <name>` (query DNS directly, bypassing `/etc/hosts`); `host <name>`.
  - DNS record types: `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias).

- [ ] **CoreDNS prerequisites** *(file 04)*
  - CoreDNS is a plugin-chain-based DNS server; configured via a **Corefile** (`. { hosts /etc/hosts }` style zone + plugin blocks).
  - The `hosts` plugin serves DNS answers straight from a static `/etc/hosts`-style file — conceptually the precursor to the in-cluster `kubernetes` plugin (file 20), which sources records from the API server instead.

- [ ] **Network Namespaces** *(file 05)*
  - `ip netns add <ns>` creates an isolated network stack (own interfaces, routes, ARP table — verified empty via `ip netns exec <ns> route`/`arp`).
  - `ip link add <a> type veth peer name <b>` creates a **veth pair**; move one end into a namespace with `ip link set <b> netns <ns>`.
  - A Linux **bridge** (`ip link add <name> type bridge`, then `ip link set <iface> master <bridge>`) connects multiple namespaces' veth ends together, acting like a virtual switch — the conceptual basis for both Docker's `docker0` bridge and a Kubernetes node's CNI bridge (`cni0`/`v-net-0`).
  - Namespaces reach external networks via routes pointing at the bridge/host, plus `iptables -t nat -A POSTROUTING -s <subnet> -j MASQUERADE` (outbound SNAT) and `iptables -t nat -A PREROUTING --dport <p> --to-destination <ip>:<p> -j DNAT` (inbound port-forwarding).

- [ ] **Docker Networking** *(file 06)*
  - Three modes: **`none`** (fully isolated, no networking), **`bridge`** (default — private network via `docker0`, containers get an internal IP, reach outside via NAT MASQUERADE; exposed via `-p hostPort:containerPort`, implemented as iptables DNAT), **`host`** (container shares the host's network namespace directly, no isolation).
  - `docker network ls`; `ip link show docker0` / `ip addr show docker0` inspect the bridge; `ip -n <container-netns-id> link/addr` inspects the container's own veth end (found via `docker inspect`).

- [ ] **CNI (Container Network Interface)** *(file 07)*
  - A **standard/contract** (not Kubernetes-specific): plugin binaries live in `/opt/cni/bin` (`bridge`, `loopback`, `host-local`, `dhcp`, `macvlan`, `portmap`, etc.), invoked by the runtime with verbs like `ADD`/`DEL`.
  - Docker does **not** natively implement CNI — Kubernetes' **kubelet** invokes the configured CNI plugin directly, bypassing Docker's own networking model.
  - Third-party CNI-compliant plugins: Weave, Calico, Flannel, Cilium.

- [ ] **Cluster Networking (host prerequisites/ports)** *(file 08)*
  - `hostname` / `hostnamectl set-hostname <name>` (+ `exec bash` to refresh the shell) — every node needs a **unique** hostname.
  - `ip a` to view IPs; `netstat -nltp` to view listening ports (`-n` numeric, `-l` listening, `-t` TCP, `-p` process name).
  - Key ports: **6443** (kube-apiserver), **2379–2380** (etcd client/peer), **10250** (kubelet), **10259** (kube-scheduler), **10257** (kube-controller-manager), **10256** (kube-proxy), **30000–32767** (NodePort range).

- [ ] **Practice Test — Explore Env** *(file 09)*
  - `ip a | grep -B2 <ip>` finds an interface by its known IP; a node's `veth*` interfaces trace back to their parent bridge (`cni0`) via `ip link show`.
  - `ip route show default` reveals the default gateway; `netstat -nplt | grep <process>` and `netstat -anp | grep <process>` find listening ports and connection counts per control-plane component (e.g. confirming etcd's client port `2379` has far more connections than its peer port `2380` on a single-control-plane cluster).

- [ ] **Pod Networking (manual bridge/route build)** *(file 10)*
  - Per node: `ip link add v-net-0 type bridge` → `ip link set dev v-net-0 up` → `ip addr add <subnet-gateway-ip>/24 dev v-net-0` — each node's bridge becomes the gateway for its own `/24` slice of a shared supernet (e.g. `10.244.0.0/16` split into `.1.0/24`, `.2.0/24`, `.3.0/24` per node).
  - Cross-node Pod reachability requires **static routes** on every node pointing at every *other* node's real LAN IP as the next-hop for that node's Pod subnet (`ip route add <remote-pod-subnet> via <remote-node-real-ip>`) — an **N×(N−1)** scaling problem that motivates CNI plugins automating this.
  - The CNI plugin contract in pseudocode: on `ADD`, create a veth pair, attach it to the bridge, assign an IP, bring the interface up; on `DEL`, delete the veth pair.

- [ ] **CNI in Kubernetes** *(file 11)*
  - kubelet flags: `--network-plugin=cni`, `--cni-bin-dir=/opt/cni/bin`, `--cni-conf-dir=/etc/cni/net.d` — check these first for "no network configured" Pod issues.
  - `/etc/cni/net.d/<N>-<name>.conf` JSON fields: `type` (which `/opt/cni/bin` binary to run, e.g. `bridge`), `bridge` (bridge name, e.g. `cni0`), `isGateway`, `ipMasq`, `ipam.type` (`host-local` = local static-range allocation), `ipam.subnet`, `ipam.routes` (installs the Pod's default route).

- [ ] **Weave Net (CNI plugin)** *(files 12–14, 16)*
  - Installed via `kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=..."`; creates a ServiceAccount, RBAC, and — critically — a **DaemonSet** (`weave-net`, one Pod per node, `2/2` containers: `weave` + `weave-npc`).
  - `kubectl logs <weave-net-pod> weave -n kube-system` (container name required — multi-container Pod).
  - `kubectl exec <pod> -- ip route` inside a Pod shows `default via <bridge-gateway-ip> dev eth0` — proof of the bridge/gateway/default-route pattern, now automated.
  - CNI plugin identification: check `kubectl get pods -n kube-system` for plugin-named Pods, and/or `ls /opt/cni/bin` for the plugin binary (e.g. `flannel`); note `bridge` is a mechanism, **not** a network provider.
  - Classic troubleshooting chain for a Pod stuck `ContainerCreating`: `kubectl get pods` → `kubectl describe pod <name>` (Events show `No network configured`/sandbox failure) → check for a running CNI DaemonSet + `/etc/cni/net.d`/`/opt/cni/bin` → `kubectl apply -f` the missing CNI manifest.

- [ ] **Weave IPAM (default range)** *(file 15)*
  - Weave Net's default Pod IP range is **`10.32.0.0/12`**, allocated via a **decentralized, gossip-based** consensus scheme across peers (not a static per-node `/24`) — configurable via `IPALLOC_RANGE`.

- [ ] **Service Networking** *(files 17–18)*
  - **ClusterIP** (default) vs **NodePort** (`PORT(S)` column shows `<port>:<nodePort>/TCP`, nodePort from the `30000–32767` range).
  - Every Service gets a `CLUSTER-IP` from `--service-cluster-ip-range` (check via `ps aux | grep kube-apiserver`, e.g. `10.96.0.0/12`); the built-in `kubernetes` Service is always the first IP in that range.
  - `kube-proxy` (a **DaemonSet**, one per node) programs `iptables` NAT rules: `KUBE-SVC-<hash>` (matches ClusterIP:port) → one or more `KUBE-SEP-<hash>` ("Service EndPoint", one per backing Pod, equal-probability `statistic --mode random` selection = the load-balancing mechanism) → `DNAT` to the real Pod IP:port; `KUBE-MARK-MASQ` handles SNAT/masquerade for cross-boundary traffic. (Alternative mode: **IPVS**.)
  - Three CIDR ranges never to confuse: **node network** (`kubectl get nodes -o wide` → `INTERNAL-IP`), **Pod network** (`kubectl get pods -A -o wide`, excluding host-network control-plane Pods), **Service network** (`kubectl get service -A` → `CLUSTER-IP`).
  - `kubectl logs -n kube-system <kube-proxy-pod>` reveals which proxy mode (`iptables`/`ipvs`) is active.

- [ ] **DNS in Kubernetes** *(file 19)*
  - **Pod DNS record** (not always enabled): `<pod-ip-with-dashes>.<namespace>.pod.cluster.local` (e.g. `10-244-1-3.apps.pod.cluster.local`).
  - **Service DNS record** (always relied upon): `<service-name>.<namespace>.svc.cluster.local` — resolves to the Service's `CLUSTER-IP`, not any individual Pod IP.
  - Cross-namespace lookups need the fully-qualified form; same-namespace lookups can use the bare short name (per the `search` list, file 20).

- [ ] **CoreDNS in Kubernetes** *(file 20)*
  - Deployed as a **Deployment** (not a DaemonSet — typically 2 replicas) + a **Service** named `kube-dns` (ClusterIP, e.g. `10.96.0.10`, ports `53/UDP,53/TCP,9153/TCP`) in `kube-system`.
  - Corefile lives in a **ConfigMap** named `coredns`; plugin chain: `errors`, `health`, `ready`, **`kubernetes <zone> { pods insecure; fallthrough ...; ttl 30 }`** (the plugin that actually answers `svc.cluster.local`/`pod.cluster.local` from the API server), `prometheus`, **`forward . /etc/resolv.conf`** (upstream/external forwarding — the "stubDomain" mechanism), `cache`, `loop`, `reload` (auto-picks-up ConfigMap edits).
  - kubelet's `/var/lib/kubelet/config.yaml` sets `clusterDNS: [<coredns-svc-ip>]` and `clusterDomain: cluster.local`, which is what populates every Pod's `/etc/resolv.conf`: `nameserver <coredns-ip>`, `search <ns>.svc.cluster.local svc.cluster.local cluster.local`, `options ndots:5` (names with fewer than 5 dots try the search suffixes first — a real perf gotcha for external lookups).
  - `host <name>` demonstrates progressive FQDN expansion via the search list.

- [ ] **Practice Test — CoreDNS in Kubernetes** *(file 21)*
  - Namespace-migration troubleshooting pattern: when a Service moves to a new namespace, any consumer referencing it by a **bare/short name** breaks; fix by updating the reference to the **namespace-qualified** short form (`<service>.<namespace>`, e.g. changing an env var from `mysql` to `mysql.payroll`), then verify with `nslookup` from the consuming Pod.

- [ ] **Ingress Controller & Ingress Resources** *(files 22–26)*
  - Ingress needs a deployed **Ingress Controller** (ConfigMap + ServiceAccount(s) + RBAC + Deployment + a fronting `NodePort`/`LoadBalancer` Service) — plain Kubernetes ships with **no** controller by default.
  - `Ingress` fields: `spec.rules[].host` (optional, `*`/omitted = all hosts), `spec.rules[].http.paths[].path`, `.pathType` (`Exact`/`Prefix`/`ImplementationSpecific` — **required**, no default, on `networking.k8s.io/v1`), `.backend`.
  - **Backend schema differs by API version**: legacy `extensions/v1beta1` → `backend.serviceName` + `backend.servicePort`; current `networking.k8s.io/v1` → `backend.service.name` + `backend.service.port.number`. Know both.
  - A bare `spec.backend` (no `rules`) = catch-all default backend for the whole Ingress.
  - **Path-based routing**: one host (or `*`), multiple paths, each to a different Service. **Host-based routing**: multiple `rules`, each keyed by a different `host`.
  - `kubectl describe ingress <name>` shows `Default backend:` (controller-wide 404 fallback, e.g. `default-http-backend:80`) and a `Rules` table (`Host`/`Path`/`Backends`).
  - `kubectl get ingress` columns: `CLASS` (`<none>` if no `IngressClass`/`ingressClassName` set), `HOSTS`, `ADDRESS`, `PORTS`.
  - **IngressClass**: cluster-scoped object identifying a controller; select via `spec.ingressClassName` (current) or `kubernetes.io/ingress.class` annotation (deprecated); mark default via `ingressclass.kubernetes.io/is-default-class: "true"`; `--watch-ingress-without-class=true` on a controller makes it also handle classless Ingresses.
  - Ingress→Service references are **namespace-scoped** — an Ingress can only route to Services in its **own** namespace.
  - **`nginx.ingress.kubernetes.io/rewrite-target: /`** — rewrites the matched path prefix to the target before proxying, so a backend that only serves `/` can be exposed under an external prefix (e.g. `/pay`) without 404ing. Often paired with `nginx.ingress.kubernetes.io/ssl-redirect: "false"` in labs with no valid TLS cert.
  - `kubectl edit ingress <name> -n <ns>` is the standard way to add/change paths in place.
  - Building an `ingress-nginx` controller from raw manifests is a realistic exam task: namespace → ConfigMap → ServiceAccount(s) (`ingress-nginx` for the controller, `ingress-nginx-admission` for its validating webhook) → RBAC → Deployment (`--configmap=`, `--default-backend-service=`, `--ingress-class=`, `--controller-class=`, `--watch-ingress-without-class=true`, `--validating-webhook*` args; `securityContext.capabilities.add: [NET_BIND_SERVICE]` to bind privileged ports as non-root) → fronting `NodePort`/`LoadBalancer` Service.
  - Common deliberately-injected YAML bugs to watch for: wrong `namespace:`, bad indentation, wrong object `name:`, wrong field-name **casing** (`nodeport` instead of `nodePort` — Kubernetes field names are case-sensitive).
  - Always verify actual namespace/Service names in the live cluster (`kubectl get ns`, `kubectl get svc -A`) rather than assuming lecture-example names match exactly.
  - File 26 (Download Presentation Deck) is a placeholder link-only page — no technical content.

---

*End of Networking notes.*
