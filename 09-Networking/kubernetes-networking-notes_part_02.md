# Networking — Complete Notes (Part 2 of 4)

> **Part 2 of 4** — covers Sections 08–14 (Cluster Networking, Practice Test — Explore
> Env, Pod Networking, CNI in Kubernetes, CNI Weave, Practice Test — CNI Weave, Practice
> Test — Deploy Network Solution).
> Previous: [kubernetes-networking-notes_part_01.md](kubernetes-networking-notes_part_01.md)
> (Sections 01–07).
> Next: [kubernetes-networking-notes_part_03.md](kubernetes-networking-notes_part_03.md)
> (Sections 15–21).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/09-Networking/`

---

## 08. Cluster Networking

- Video reference: *Pre-requisite Cluster Networking* — https://kodekloud.com/topic/cluster-networking/
- Topic: this is a **pre-requisite** lesson — before diving into how Pod networking and CNI plugins work, you need to be comfortable with basic host-level networking commands, because troubleshooting cluster networking issues on the exam starts at this level (hostname, IP address, listening ports).
- The lesson lists exactly 3 pre-requisite skills for cluster networking:
  - **Set the unique hostname.**
  - **Get the IP addr of the system (master and worker node).**
  - **Check the Ports.**

### IP and Hostname

- To view the hostname:
  ```
  $ hostname 
  ```
- To view the IP addr of the system:
  ```
  $ ip a
  ```
  - `ip a` (short for `ip addr`) lists every network interface on the host along with its assigned IP address(es), MAC address, and state — this is the fundamental command used throughout the rest of the Networking section (files 09 and 10) to identify interfaces like `eth0`, `veth*`, `cni0`, and later the custom bridge `v-net-0`.

### Set the hostname

- Every node in a Kubernetes cluster needs a **unique hostname** (the cluster uses the hostname as the Node object's name by default), so this command is provided to fix a duplicate/default hostname on a node:
  ```
  $ hostnamectl set-hostname <host-name>
  
  $ exec bash
  ```
  - `hostnamectl set-hostname <host-name>` permanently changes the system's hostname.
  - `exec bash` restarts the current shell so the new hostname is reflected immediately in the shell prompt (without this, the old hostname may still show in the current terminal session even though the underlying system hostname has changed).

### View the Listening Ports of the system

```
$ netstat -nltp
```

- `-n` — show IP addresses/ports numerically (skip DNS/service-name resolution).
- `-l` — show only listening sockets.
- `-t` — show only TCP sockets.
- `-p` — show the PID/process name owning the socket.
- **Exam-relevant:** this is the go-to command for verifying which control-plane components are actually listening where on a node — e.g. confirming `kube-apiserver` is bound to `6443`, `etcd` to `2379`/`2380`, `kube-scheduler` to `10259`, `kube-controller-manager` to `10257`, and `kubelet` to `10250` — exactly the technique the next file's practice test builds on with `netstat -nplt` and `netstat -anp`.
- **Exam-relevant reference (general Kubernetes/kubeadm port-requirements knowledge — not spelled out as a table in this source file, but is what the "Check Required Ports" reference link below documents):** a kubeadm cluster requires roughly this inbound port set to be open between nodes:
  - **Control-plane node(s):** `6443` (kube-apiserver), `2379-2380` (etcd client/peer API), `10250` (kubelet API), `10259` (kube-scheduler), `10257` (kube-controller-manager).
  - **Worker node(s):** `10250` (kubelet API), `10256` (kube-proxy), `30000-32767` (NodePort Services range).

#### References Docs

- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports
- https://kubernetes.io/docs/concepts/cluster-administration/networking/

---

## 09. Practice Test — Explore Env

Solutions to the practice test — Explore Environment. Step-by-step lab-guide walkthrough:

1. **"How many nodes are part of this cluster?"**
   ```
   kubectl get nodes
   ```
   - Count the rows returned in the output.

2. **"What is the Internal IP address of the controlplane node in this cluster?"**
   ```
   kubectl get nodes -o wide
   ```
   - Note the value in the `INTERNAL-IP` column for the `controlplane` row.

3. **"What is the network interface configured for cluster connectivity on the controlplane node?"**
   - This will be the network interface that has the same IP address determined in the previous question.
   ```
   ip a
   ```
   - There is quite a lot of output for this command, so filter it better:
     ```
     ip a | grep -B2 X.X.X.X
     ```
     where `X.X.X.X` is the IP address obtained from the previous question. `grep -B2` finds the line containing the value being searched for and prints that line plus the previous 2 lines of output. Sample output (values differ every run):
     ```
     3058: eth0@if3059: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
        link/ether 02:42:c0:08:ea:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 192.8.234.3/24 brd 192.8.234.255 scope global eth0
     ```
   - **Answer:** `eth0`

4. **"What is the MAC address of the interface on the controlplane node?"**
   - This value is also present in the output of the command run for the previous question. The MAC address is the value in the `link/ether` field and is 6 hex numbers separated by `:` (the value differs every time the lab runs). Using the same sample output as above:
     ```
     3058: eth0@if3059: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
        link/ether 02:42:c0:08:ea:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 192.8.234.3/24 brd 192.8.234.255 scope global eth0
     ```
   - **Answer:** `02:42:c0:08:ea:03`

5. **"What is the IP address assigned to node01?"**
   ```
   kubectl get nodes -o wide
   ```
   - Note the value in the `INTERNAL-IP` column for `node01`.

6. **"What is the MAC address assigned to node01?"**
   - For this, SSH onto `node01` to view its interfaces. The IP to look for is already known from the previous question:
     ```
     ssh node01
     ip a | grep -B2 X.X.X.X
     ```
     where `X.X.X.X` is the IP address obtained from the previous question. Again, look at the `link/ether` field.
   - It could be guessed that the correct interface on `node01` is also `eth0`, and simply run:
     ```
     ip link show eth0
     ```
     but it's best to be sure (i.e. confirm via the `grep -B2` approach rather than assuming).
   - Return to `controlplane`:
     ```
     exit
     ```

7. **"We use Containerd as our container runtime. What is the interface/bridge created by Containerd on this host?"**
   - This is not immediately straightforward.
     ```
     ip link show
     ```
   - Know that:
     - Any interface with name beginning **`eth`** is a "physical" interface, and represents a network card attached to the host.
     - Interface **`lo`** is the loopback, and covers all IP addresses starting with `127`. Every computer has this.
     - Any interface with name beginning **`veth`** is a virtual network interface used for tunnelling between the host and the pod network. These connect with bridges, and the bridge interface name is listed with their details.
   - It can be seen that for the two `veth` devices, they are associated with another device in the list, `cni0` — therefore that is the answer.
   - **Answer:** `cni0`

8. **"What is the state of the interface cni0?"**
   - Visible in the output of the previous command — the state field for `cni0` is:
   - **Answer:** `UP`

9. **"If you were to ping google from the controlplane node, which route does it take? What is the IP address of the Default Gateway?"**
   ```
   ip route show default
   ```
   - Note the output — it reveals the default gateway IP address that outbound (e.g. internet-bound) traffic from the controlplane node is routed through.

10. **"What is the port the kube-scheduler is listening on in the controlplane node?"**
    - Use the [netstat](https://linux.die.net/man/8/netstat) command to look at network sockets used by programs running on the host. There's a lot of output, so filter by process name, i.e. `kube-scheduler`:
      ```
      netstat -nplt | grep kube-scheduler
      ```
      - What the `netstat` options mean:
        - `-n` — Show IP addresses (don't try to resolve to host names).
        - `-p` — Show the process names (e.g. `kube-scheduler`).
        - `-l` — Include only *listening* sockets.
        - `-t` — Include only TCP sockets.
    - Output:
      ```
      tcp        0      0 127.0.0.1:10259         0.0.0.0:*               LISTEN      3291/kube-scheduler
      ```
    - It's listening on localhost, port **`10259`**.

11. **"Notice that ETCD is listening on two ports. Which of these have more client connections established?"**
    - Use `netstat` with slightly different options and filter for `etcd`:
      ```
      netstat -anp | grep etcd
      ```
      - What the `netstat` options mean:
        - `-a` — Include sockets in all states.
        - `-n` — Show IP addresses (don't try to resolve to host names).
        - `-p` — Show the process names (e.g. `etcd`).
    - By far and away, the most used port is **`2379`**.

12. **(Information step.)** `2379` is the port of ETCD to which the API server connects. There are multiple concurrent connections so that the API server can process multiple etcd operations simultaneously. `2380` is only for etcd peer-to-peer connectivity when there are multiple controlplane nodes — in this lab there is only one, so `2380` shows no client connections.

- **Exam-relevant takeaway:** this whole practice test is a drill in *reading* raw Linux networking command output (`ip a`, `ip link show`, `ip route show default`, `netstat`) rather than running Kubernetes-specific commands — a skill the exam expects for diagnosing node/network issues without any GUI. The `grep -B2 <ip>` trick to find an interface by its known IP, and using `veth` devices to trace back to their parent bridge (`cni0`), are both directly reused in the next lesson (Pod Networking) and in real CNI troubleshooting.

---

## 10. Pod Networking

- Video reference: *Pod Networking* — https://kodekloud.com/topic/pod-networking/
- Topic: manually building the low-level networking plumbing (bridges, IP ranges, static routes) that a CNI plugin would otherwise automate — a hands-on simulation of what happens "under the hood" across a 3-node cluster so that a Pod on one node can reach a Pod on another node.
- Scenario: 3 nodes (`node01`, `node02`, `node03`), each attached to a LAN on the `192.168.1.0` subnet with node IPs `192.168.1.11`, `192.168.1.12`, `192.168.1.13` respectively (per the referenced diagram `net11.PNG`).

### Create the bridge network on each node

- To add a bridge network on each node:

  > node01
  ```
  $ ip link add v-net-0 type bridge
  ```

  > node02
  ```
  $ ip link add v-net-0 type bridge
  ```

  > node03
  ```
  $ ip link add v-net-0 type bridge
  ```
  - `ip link add v-net-0 type bridge` creates a new virtual Linux bridge interface named `v-net-0` on each node — this bridge is the local Layer-2 switch that Pod veth-pair endpoints on that node will later attach to (this is exactly the role played by `cni0` observed in file 09's practice test, just built here by hand and named `v-net-0`).

### Bring the bridge interface up

- Currently it's down; turn it up.

  > node01
  ```
  $ ip link set dev v-net-0 up
  ```

  > node02
  ```
  $ ip link set dev v-net-0 up
  ```

  > node03
  ```
  $ ip link set dev v-net-0 up
  ```

### Assign an IP address to the bridge interface

- Set the IP addr for the bridge interface — each node's bridge gets a different `/24` subnet, all carved out of a shared `10.244.0.0/16` supernet:

  > node01
  ```
  $ ip addr add 10.244.1.1/24 dev v-net-0
  ```

  > node02
  ```
  $ ip addr add 10.244.2.1/24 dev v-net-0
  ```

  > node03
  ```
  $ ip addr add 10.244.3.1/24 dev v-net-0
  ```
  - Each node's `v-net-0` bridge becomes the **gateway address** for its own local Pod subnet: `10.244.1.1` for node01's `10.244.1.0/24`, `10.244.2.1` for node02's `10.244.2.0/24`, `10.244.3.1` for node03's `10.244.3.0/24`. Individual Pod veth endpoints on each node get an IP address inside that node's `/24` (e.g. `10.244.1.2`, `10.244.1.3` on node01) and are attached to that node's bridge, so all Pods on the same node can talk to each other directly through the bridge.

- **Inferred context (image `net11.PNG`):** three node boxes (NODE1 / 192.168.1.11, NODE2 / 192.168.1.12, NODE3 / 192.168.1.13), each containing a Docker icon plus two colored container circles, and each with a dashed "BRIDGE v-net-0" cloud inside the node box. All three nodes connect down into a shared "LAN 192.168.1.0" cloud at the bottom — this is the physical/underlying network the nodes' `eth0`-style interfaces sit on, separate from (and underneath) each node's private `v-net-0` bridge network.

### Check reachability

```
$ ping 10.244.2.2
Connect: Network is unreachable
```

- Pinging a Pod IP on a *different* node (`10.244.2.2` lives on node02's bridge subnet) from node01 fails with **"Network is unreachable"** — this is expected: node01's kernel has no route telling it *how* to reach the `10.244.2.0/24` network; it only knows about its own locally-attached `10.244.1.0/24` bridge subnet and whatever route(s) it's been given for the outside world.

### Add routes in the routing table

- Add a route in the routing table so a node knows how to reach another node's Pod subnet, by routing it **via that node's real (LAN) IP address**:
  ```
  $ ip route add 10.244.2.2 via 192.168.1.12
  ```

- Every node needs a route entry for every *other* node's Pod subnet, pointed at that other node's real LAN IP (acting as the next-hop gateway):

  > node01
  ```
  $ ip route add 10.244.2.2 via 192.168.1.12
  
  $ ip route add 10.244.3.2 via 192.168.1.13
  ```

  > node02
  ```
  $ ip route add 10.244.1.2 via 192.168.1.11
  
  $ ip route add 10.244.3.2 via 192.168.1.13
  
  ```

  > node03
  ```
  $ ip route add 10.244.1.2 via 192.168.1.11
  
  $ ip route add 10.244.2.2 via 192.168.1.12
  ```
  - **Note on the pattern:** each node adds exactly 2 static routes — one for each of the *other two* nodes' Pod subnets/IPs — with the **next hop set to the other node's actual (LAN) IP address**, not to a Pod IP. This is precisely how "routed" CNI approaches (like Flannel's host-gw backend, or the general model any CNI implements) achieve cross-node Pod-to-Pod connectivity: the underlying physical network only needs to know how to get packets between node IPs; each node's kernel routing table is what stitches together "if the destination is inside subnet X, send it to node Y's real IP, and node Y's kernel/bridge will deliver it locally from there."
  - **Exam-relevant / scaling problem being illustrated:** this manual approach requires **N × (N−1)** static route entries to fully connect an N-node cluster (each node needs one route per every *other* node) — with only 3 nodes that's 6 total route lines, but this obviously does not scale to a real cluster with dozens or hundreds of nodes. This is exactly the pain point that motivates the next section ("Add a single large network") and ultimately the whole point of CNI plugins (files 11–12): they automate exactly this bridge-creation, IP-allocation, and route-programming work across every node in the cluster automatically, instead of requiring an admin to hand-run `ip route add` on every single node for every other node.

### Add a single large network

- **Inferred context (image `net12.PNG`):** the same 3-node layout as `net11.PNG`, but now annotated with the full picture: each node's bridge subnet is labeled (`10.244.1.0/24` on NODE1 gateway `10.244.1.1`, `10.244.2.0/24` on NODE2 gateway `10.244.2.1`, `10.244.3.0/24` on NODE3 gateway `10.244.3.1`), individual container/Pod IPs are shown attached to each bridge (e.g. `10.244.1.2`/`10.244.1.3` on NODE1, `10.244.2.2` on NODE2, `10.244.3.2` on NODE3), and critically the three per-node bridge clouds are drawn merged together into one continuous "**10.244.0.0/16**" supernet spanning all three nodes. Below the diagram is a **Network / Gateway** table explicitly mapping:

  | NETWORK | GATEWAY |
  |---|---|
  | 10.244.1.0/24 | 192.168.1.11 |
  | 10.244.2.0/24 | 192.168.1.12 |
  | 10.244.3.0/24 | 192.168.1.13 |

  - **Concept being conveyed:** rather than treating each node's bridge subnet as an isolated island requiring pairwise static routes, you can think of (and — via a CNI plugin — actually configure) the **entire cluster's Pod network as a single flat `10.244.0.0/16` address space**, where each node simply "owns" one `/24` slice of it, and the node's real LAN IP acts as the gateway/next-hop for reaching that slice. This single-large-network table is exactly the kind of routing information a router (or, in a real cluster, either a cloud provider's VPC routing table, or a CNI plugin's own routing/overlay mechanism) would be configured with once, instead of hand-adding routes on every node — directly foreshadowing how a real CNI plugin (e.g. Flannel, Calico, Weave) manages this automatically.

### Container Network Interface

- **Inferred context (image `net13.PNG`):** the same 3-node/single-supernet diagram as `net12.PNG` on the right, paired on the left with the Kubernetes logo, the "CONTAINER NETWORK INTERFACE (CNI)" title, and a code panel labeled **`net-script.sh`** showing the actual CNI plugin contract as pseudocode:
  ```
  net-script.sh

  ADD)
    # Create veth pair
    # Attach veth pair
    # Assign IP Address
    # Bring Up Interface
    ip -n <namespace> link set .....

  DEL)
    # Delete veth pair
    ip link del .....
  ```
  - **What this shows (this is the core CNI plugin contract):** a CNI plugin is conceptually just an executable/script that Kubernetes (via the container runtime, invoked by the kubelet) calls with a command — `ADD` or `DEL` — for a given container's network namespace:
    - On **`ADD`** (i.e. when a Pod/container is created and needs network connectivity), the script must: create a **veth pair**, **attach** one end of the veth pair to the node's bridge and the other end into the Pod's network namespace, **assign an IP address** to the Pod-side interface (from the node's Pod-subnet range), and **bring the interface up** — using a command pattern like `ip -n <namespace> link set .....`.
    - On **`DEL`** (i.e. when a Pod/container is deleted), the script must **delete the veth pair**, using a command pattern like `ip link del .....`.
  - This is exactly the manual bridge/veth/IP/route work performed step-by-step earlier in this same file (bridge creation, IP assignment, routing) — the point of this closing diagram is that a **CNI plugin automates precisely this sequence of `ip` commands**, invoked automatically by the container runtime for every Pod's `ADD`/`DEL` lifecycle event, instead of an administrator running them by hand. This directly sets up file 11 (CNI in Kubernetes), which explains *how* the kubelet/runtime knows which script/binary to invoke and where to find it.

#### References Docs

- https://kubernetes.io/docs/concepts/workloads/pods/

---

## 11. CNI in Kubernetes

- Video reference: *CNI in Kubernetes* — https://kodekloud.com/topic/cni-in-kubernetes/
- Topic: how Kubernetes actually locates and invokes a CNI plugin — closing the loop from file 10's manual `net-script.sh` ADD/DEL contract to the real kubelet configuration that wires a chosen CNI plugin into the container-creation lifecycle.

### Configuring CNI

- **Inferred context (image `net1.PNG`):** a `kubelet.service` systemd unit file excerpt showing the kubelet's `ExecStart` command line:
  ```
  kubelet.service

  ExecStart=/usr/local/bin/kubelet \
    --config=/var/lib/kubelet/kubelet-config.yaml \
    --container-runtime=remote \
    --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
    --image-pull-progress-deadline=2m \
    --kubeconfig=/var/lib/kubelet/kubeconfig \
    --network-plugin=cni \
    --cni-bin-dir=/opt/cni/bin \
    --cni-conf-dir=/etc/cni/net.d \
    --register-node=true \
    --v=2
  ```
  with the three CNI-related flags highlighted:
  - **`--network-plugin=cni`** — tells the kubelet to use the CNI networking model (as opposed to no networking, or a different legacy plugin type).
  - **`--cni-bin-dir=/opt/cni/bin`** — the directory the kubelet/container runtime searches for CNI plugin **binaries** (executables) to actually invoke for `ADD`/`DEL` operations.
  - **`--cni-conf-dir=/etc/cni/net.d`** — the directory the kubelet/container runtime searches for CNI plugin **configuration files** (JSON), which tell it *which* plugin/binary to invoke and with what settings (e.g. bridge name, subnet, IPAM config).
  - **Exam-relevant:** these two directory flags (`--cni-bin-dir` and `--cni-conf-dir`) are the two things to check first when diagnosing "Pod stuck in `ContainerCreating`, no network configured" issues (as seen later in file 14's practice test) — if the conf directory is empty or the referenced binary is missing from the bin directory, no CNI plugin can run and Pods will never get an IP.

- Check the status of the Kubelet Service:
  ```
  $ systemctl status kubelet.service
  ```

### View Kubelet Options

```
$ ps -aux | grep kubelet
```
- This lists the running `kubelet` process and its full command line (equivalent information to the `ExecStart=` line shown above, but read live off a running process rather than the unit file) — useful for confirming exactly which flags (including `--cni-bin-dir`/`--cni-conf-dir`) are actually in effect on a given node.

### Check the Supportable Plugins

- To check all the supportable plugins available in the `/opt/cni/bin` directory:
  ```
  $ ls /opt/cni/bin
  
  ```
  - This directory holds every CNI plugin **binary** installed on the node (e.g. `bridge`, `loopback`, `host-local`, `flannel`, `portmap`, etc., depending on what's installed) — this is the directory the `--cni-bin-dir` kubelet flag points at.

### Check the CNI Plugins

- To check the CNI plugin(s) which the kubelet needs to use:
  ```
  ls /etc/cni/net.d
  
  ```
  - This directory holds the CNI plugin **configuration file(s)** — this is the directory the `--cni-conf-dir` kubelet flag points at. The kubelet/runtime reads the configuration file(s) here to determine which plugin binary (from `/opt/cni/bin`) to actually invoke, and with what parameters.

### Format of Configuration File

- **Inferred context (image `net2.PNG`):** a "View kubelet options" panel demonstrating the actual conf-dir contents and format:
  ```
  $ ls /etc/cni/net.d
  10-bridge.conf

  $ cat /etc/cni/net.d/10-bridge.conf
  {
      "cniVersion": "0.2.0",
      "name": "mynet",
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
          "type": "host-local",
          "subnet": "10.22.0.0/16",
          "routes": [
              { "dst": "0.0.0.0/0" }
          ]
      }
  }
  ```
  - **Field-by-field meaning of this CNI configuration JSON:**
    - **`cniVersion`** — the CNI spec version this config targets (`"0.2.0"`).
    - **`name`** — the logical network name (`"mynet"`) — this is the network identity that gets attached to each Pod's `ADD` call.
    - **`type`** — which plugin **binary** (from `/opt/cni/bin`, e.g. `bridge`) the runtime should actually execute to fulfill `ADD`/`DEL` requests for this network.
    - **`bridge`** — the name of the Linux bridge to create/use (`"cni0"`) — this is exactly the `cni0` bridge that was observed in file 09's practice test (the bridge that all the node's `veth` pairs attach to), and plays the identical conceptual role as the hand-created `v-net-0` bridge in file 10.
    - **`isGateway: true`** — the bridge itself is assigned an IP and acts as the default gateway for Pods on this node (mirroring `ip addr add 10.244.1.1/24 dev v-net-0` from file 10).
    - **`ipMasq: true`** — enables IP masquerading (SNAT) for traffic leaving this bridge network, so Pod-sourced traffic going to destinations outside the Pod network gets rewritten to use the node's own IP (needed for Pods to reach the outside world/internet through the node).
    - **`ipam`** — the **IP Address Management** sub-plugin configuration:
      - **`type: "host-local"`** — use the `host-local` IPAM plugin, which allocates IPs from a locally-configured range and persists allocation state on the local node's disk (as opposed to querying a central/external IPAM service).
      - **`subnet: "10.22.0.0/16"`** — the IP range this node's IPAM plugin allocates Pod IPs from.
      - **`routes: [ { "dst": "0.0.0.0/0" } ]`** — installs a default route (`0.0.0.0/0`, i.e. "everything") inside each Pod's network namespace pointing back out through the bridge gateway — this is what makes the earlier `kubectl exec test -- ip route` output in file 12 (`default via 10.244.1.1 dev eth0`) show a default route through the node's bridge IP.
  - This configuration file is exactly what a plugin binary like `bridge` (invoked by the container runtime per file 10's `net-script.sh` `ADD`/`DEL` contract) reads in order to know *how* to create the veth pair, which bridge to attach it to, and which IPAM sub-plugin to delegate address allocation to.

#### References Docs

- https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
- https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/

---

## 12. CNI Weave

- Video reference: *CNI Weave* — https://kodekloud.com/topic/cni-weave/
- Topic: **Weave Net**, a concrete real-world CNI plugin, deployed as the cluster's actual networking solution (replacing the manual bridge/route work of file 10 and the generic `bridge`-type CNI config of file 11 with a production-grade overlay network implementation).

### Deploy Weave

- Installing [weave net](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/) onto the Kubernetes cluster with a single command:
  ```
  $ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  serviceaccount/weave-net created
  clusterrole.rbac.authorization.k8s.io/weave-net created
  clusterrolebinding.rbac.authorization.k8s.io/weave-net created
  role.rbac.authorization.k8s.io/weave-net created
  rolebinding.rbac.authorization.k8s.io/weave-net created
  daemonset.apps/weave-net created
  ```
  - The URL is dynamically parameterized with `$(kubectl version | base64 | tr -d '\n')` — this base64-encodes the local `kubectl version` output and passes it as the `k8s-version` query parameter, letting the Weave Cloud endpoint serve back a manifest tailored to the cluster's actual Kubernetes version.
  - The manifest this URL returns creates, in order: a **ServiceAccount** (`weave-net`), a **ClusterRole** + **ClusterRoleBinding** (cluster-wide RBAC permissions Weave needs, e.g. to watch Nodes/Pods), a namespaced **Role** + **RoleBinding**, and — most importantly — a **DaemonSet** (`weave-net`), which ensures exactly one Weave Net Pod runs on **every** node in the cluster (this is precisely the deployment model a CNI networking overlay needs: every node must run the plugin's agent so every node can participate in the overlay mesh).
  - **Exam-relevant:** the `daemonset.apps/weave-net` object is the actual workload doing the networking — knowing that Weave (like most CNI network plugins: Calico, Flannel, Cilium) is deployed as a **DaemonSet** in `kube-system` is a core piece of exam knowledge for diagnosing "why are Pods stuck in `ContainerCreating`" issues (check whether the network DaemonSet's Pods are actually `Running` on every node).

### Weave Peers

```
$ kubectl get pods -n kube-system
NAME                                      READY   STATUS             RESTARTS   AGE
coredns-66bff467f8-894jf                  1/1     Running            0          52m
coredns-66bff467f8-nck5f                  1/1     Running            0          52m
etcd-controlplane                         1/1     Running            0          52m
kube-apiserver-controlplane               1/1     Running            0          52m
kube-controller-manager-controlplane      1/1     Running            0          52m
kube-keepalived-vip-mbr7d                 1/1     Running            0          52m
kube-proxy-p2mld                          1/1     Running            0          52m
kube-proxy-vjcwp                          1/1     Running            0          52m
kube-scheduler-controlplane               1/1     Running            0          52m
weave-net-jgr8x                           2/2     Running            0          45m
weave-net-tb9tz                           2/2     Running            0          45m
```
- Two `weave-net-*` Pods are visible — one per node in this cluster (a controlplane node and presumably one worker), confirming the DaemonSet has successfully scheduled a **Weave peer** onto every node so they can mesh together into the overlay network. Note each weave-net Pod shows **`2/2`** containers ready — a Weave Net Pod runs **two** containers: the `weave` container (the actual network router/overlay agent) and a `weave-npc` container (Weave's Network Policy Controller, enforcing NetworkPolicy objects).

### View the logs of Weave Pod's

```
$ kubectl logs weave-net-tb9tz weave -n kube-system 
```
- Since a weave-net Pod has multiple containers, the container name (`weave`) must be specified explicitly after the Pod name to view that specific container's logs (as opposed to the `weave-npc` container) — this is the primary troubleshooting command for diagnosing Weave-specific networking problems (e.g. peer connection failures, IPAM range conflicts).

### View the default route in the Pod

```
$ kubectl run test --image=busybox --command -- sleep 4500
pod/test created

$ kubectl exec test -- ip route
default via 10.244.1.1 dev eth0
```
- `kubectl run test --image=busybox --command -- sleep 4500` creates a simple long-lived test Pod (`busybox` sleeping for 4500 seconds) purely so there's a Pod to `exec` into and inspect.
- `kubectl exec test -- ip route` runs `ip route` **inside** the Pod's network namespace, printing its routing table.
- Output `default via 10.244.1.1 dev eth0` shows the Pod's default route pointing at `10.244.1.1` (the node's Weave bridge/gateway IP) via its `eth0` interface (the Pod-side end of the CNI-created veth pair) — this is the live, real-world confirmation of exactly the same concept manually built in file 10 (`ip addr add 10.244.1.1/24 dev v-net-0` as the bridge gateway) and configured declaratively in file 11's `10-bridge.conf` (`"isGateway": true` plus the IPAM `routes: [{ "dst": "0.0.0.0/0" }]` entry) — Weave Net is simply the production implementation automating that identical bridge/gateway/default-route pattern across every node in the cluster, using its own overlay network addressing (here, the `10.244.0.0/16`-style range Weave allocates by default).

#### References Docs

- https://kubernetes.io/docs/concepts/cluster-administration/addons/
- https://www.weave.works/docs/net/latest/kubernetes/kube-addon/

---

## 13. Practice Test — CNI Weave

Solutions to the practice test — CNI Weave. Step-by-step lab-guide walkthrough:

1. **"Inspect the kubelet service and identify the container runtime value is set for Kubernetes."**
   - Check the kubelet unit file:
     ```bash
     systemctl cat kubelet
     ```
   - Note this line from the output:
     ```
     EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
     ```
   - Inspect this file:
     ```bash
     cat /var/lib/kubelet/kubeadm-flags.env
     ```
   - The answer can be found as the value of `--container-runtime`.
   - **Answer:** `REMOTE`

2. **"What is the path configured with all binaries of CNI supported plugins?"**
   - This is the standard location for the installation of CNI plugins.
   - **Answer:** `/opt/cni/bin`

3. **"Identify which of the below plugins is not available in the list of available CNI plugins on this host?"**
   ```bash
   ls -l /opt/cni/bin
   ```
   - Find the option from the given answer choices that does **not** appear in the output of the above command.
   - **Answer:** `cisco`

4. **"What is the CNI plugin configured to be used on this kubernetes cluster?"**
   - From the available answer options, first work out which of the four is not the name of a container networking provider at all. Of the three that remain, only one of them is actually present in `/opt/cni/bin`.
   - **Answer:** `flannel`
   - Note: `bridge` is a mechanism/plugin type for connecting networks together, **not** a network *provider* — a subtlety the wrong-answer options are testing.

5. **"What binary executable file will be run by kubelet after a container and its associated namespace are created."**
   - Following on from Q4:
   - **Answer:** `flannel`
   - All the files in `/opt/cni/bin` are binary executables with tasks related to configuring network namespaces. After the network namespace is configured using the other programs, `flannel` implements the actual network (attaches the Pod to the overlay/network fabric).
   - Reference article on what the programs in `/opt/cni/bin` are for: https://tonylixu.medium.com/k8s-network-cni-introduction-b035d42ad68f

- **Exam-relevant takeaway:** this whole practice test is a drill in reading the kubelet's actual runtime configuration end-to-end — `systemctl cat kubelet` to find the `EnvironmentFile`, `cat`-ing that env file to find `--container-runtime`, and then `ls /opt/cni/bin` / `ls /etc/cni/net.d` to determine which specific CNI plugin binary and config are actually wired up on a real running node — directly exercising the theory covered in file 11.

---

## 14. Practice Test — Deploy Network Solution

Solutions to the practice test — Deploy Networking Solution. Step-by-step lab-guide walkthrough:

1. **"We have deployed an application called app in the default namespace. What is the state of the pod?"**
   ```bash
   kubectl get pods
   ```
   - Note it is stuck at `ContainerCreating`. It will remain this way (indefinitely, until the underlying cause is fixed).
   - **Answer:** `NotRunning`

2. **"Inspect why the POD is not running."**
   ```bash
   kubectl describe pod app
   ```
   - The answer is in the `Events` section. It cannot allocate an IP address, therefore:
   - **Answer:** `No network configured`
   - **Exam-relevant:** this is the single most common symptom/root-cause pairing tested for CNI issues on the CKA exam — a Pod permanently stuck in `ContainerCreating`, with `kubectl describe pod` showing a `FailedCreatePodSandBox`/"failed to set up sandbox container" style event mentioning it could not allocate an IP or find a CNI network — and the fix is precisely what this file's next step demonstrates: no CNI plugin/network solution has actually been deployed to the cluster yet (an empty or missing `/etc/cni/net.d`, or no CNI DaemonSet running), so the container runtime has nothing to invoke for the Pod's `ADD` operation from file 10/11's contract.

3. **"Deploy weave-net networking solution to the cluster."**
   - Apply the manifest found under the `/root/weave` directory (i.e. `kubectl apply -f /root/weave/<manifest-file>`, deploying the pre-staged Weave Net manifest, which — per file 12 — creates the `weave-net` ServiceAccount, RBAC objects, and DaemonSet, which then runs a Weave peer Pod on every node and finally allows the sandbox for the earlier `app` Pod to be created successfully, unblocking it out of `ContainerCreating`).

- **Exam-relevant takeaway (ties files 10–14 together):** the full troubleshooting chain for "no network configured" is: (1) `kubectl get pods` to spot the stuck `ContainerCreating` state → (2) `kubectl describe pod <name>` to read the `Events` section and confirm it's an IP-allocation/sandbox-creation failure → (3) check for a running CNI networking DaemonSet (`kubectl get pods -n kube-system` looking for `weave-net`/`flannel`/`calico`-style Pods, per file 12) and check `/etc/cni/net.d` and `/opt/cni/bin` on the affected node (per file 11) → (4) if no network solution is deployed at all, `kubectl apply -f` the appropriate CNI manifest (here, a pre-staged Weave manifest under `/root/weave`) to resolve it.

---

*Continued in [kubernetes-networking-notes_part_03.md](kubernetes-networking-notes_part_03.md)
— Sections 15–21 (Weave IPAM, Practice Test — Networking Weave, Service Networking,
Practice Test — Service Networking, DNS in Kubernetes, CoreDNS in Kubernetes, Practice
Test — CoreDNS in Kubernetes).*
