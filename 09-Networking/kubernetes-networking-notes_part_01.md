# Networking — Complete Notes (Part 1 of 5)

> **Part 1 of 5** — covers Sections 01–07 (Section Introduction, Pre-requisite Switching/
> Routing/Gateways, Pre-requisite DNS, Pre-requisite CoreDNS, Pre-requisite Network
> Namespace, Pre-requisite Docker Networking, Pre-requisite CNI).
> Next: [kubernetes-networking-notes_part_02.md](kubernetes-networking-notes_part_02.md)
> (Sections 08–14).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/09-Networking/`

---

## 01. Networking — Section Introduction

- Video reference: *Networking Section Introduction* — https://kodekloud.com/topic/networking-introduction/
- This is a short scene-setting/table-of-contents file with no diagrams or commands of its own — it simply previews the full structure of the Networking section:
  - **Prerequisite** topics (foundational Linux/Docker networking knowledge needed before Kubernetes-specific networking makes sense):
    - **Switching, Routing and Gateways**
      - Switching
      - Routing
      - Default Gateway
    - **DNS**
      - DNS Configuration on Linux
    - **CoreDNS**
    - **Network Namespace**
    - **Docker Networking**
    - **CNI** (Container Network Interface)
  - **Cluster Networking** (Kubernetes-specific, covered later in the source section)
  - **Pod Networking**
  - **CNI in Kubernetes**
  - **CoreDNS** (in a Kubernetes context)
  - **Ingress**
- **Note:** structurally identical in purpose to other short "introduction" pages seen elsewhere in this course — a pure outline page with no technical content of its own to extract. The remainder of this document (files 02–07) covers the **Prerequisite** block in full detail: general Linux networking fundamentals (switching/routing/gateways, DNS, CoreDNS basics, network namespaces, Docker networking, and the CNI spec) that underpin how Kubernetes wires up Pod-to-Pod and Pod-to-outside-world networking later in the course.

---

## 02. Pre-requisite — Switching, Routing and Gateways

- Video/lecture reference: https://kodekloud.com/topic/pre-requisite-switching-routing-gateways-cni-in-kubernetes/
- Topic: the foundational Linux networking building blocks — switching (Layer 2, local connectivity), routing (Layer 3, cross-network connectivity), and default gateways (routing to "everywhere else") — that everything later in the Networking section (network namespaces, Docker networking, CNI, Kubernetes Pod networking) is built on top of.

### Switching

- **Switching** is about connecting hosts together **on the same network/subnet** — a switch operates at Layer 2 and simply forwards frames between hosts that share the same IP subnet, with no routing decision involved.
- **To see the interfaces on the host system:**
  ```
  $ ip link
  ```
- **To see the IP address interfaces:**
  ```
  $ ip addr
  ```
- **Inferred context (image `net14.PNG`):** based on the surrounding narrative (switching = same-subnet connectivity) and the fact that this is the very first diagram in the Networking section, this almost certainly shows a simple **two (or three) host LAN diagram connected via a switch** — e.g. Host A (`192.168.1.10`) and Host B (`192.168.1.11`) both plugged into the same Layer-2 switch, each with a network interface (`eth0`) carrying an IP address in the same `192.168.1.0/24` subnet — illustrating that hosts on the same subnet can talk directly to one another through a switch without needing any routing, and that `ip link`/`ip addr` are the commands used to inspect each host's own view of its interface(s) in that topology.

### Routing

- **Routing** is about connecting hosts that are on **different networks/subnets** — this requires a device (a router) that knows how to forward packets from one subnet to another, and a routing table on each host telling it which "next hop" to use to reach a given destination network.
- **To see the existing routing table on the host system:**
  ```
  $ route
  ```
  ```
  $ ip route show
  or
  $ ip route list
  ```
- **To add entries into the routing table:**
  ```
  $ ip route add 192.168.1.0/24 via 192.168.2.1
  ```
  - This adds a route saying: "to reach any host in the `192.168.1.0/24` network, send the packet via the next-hop router at `192.168.2.1`."
- **Inferred context (image `net15.PNG`):** likely shows a **two-subnet topology joined by a router** — e.g. Network A (`192.168.1.0/24`, containing Host A at `192.168.1.10`) and Network B (`192.168.2.0/24`, containing Host B at `192.168.2.10`), connected by a router that has an interface/IP in *each* subnet (e.g. `192.168.1.1` on the Network A side and `192.168.2.1` on the Network B side) — alongside a sample **routing table** (Destination / Gateway / Genmask / Iface columns, matching `route`'s real output format) showing the entry added by the `ip route add 192.168.1.0/24 via 192.168.2.1` command above, i.e. how Host B (on Network B) would route traffic destined for Network A through the router at `192.168.2.1`.

### Gateways

- A **default gateway** (or "default route") is the catch-all next-hop used when a packet's destination doesn't match any more-specific route in the routing table — i.e. "for anywhere else I don't have an explicit route for, send it here."
- **To add a default route:**
  ```
  $ ip route add default via 192.168.2.1
  ```
- **To check whether IP forwarding is enabled on the host** (required on any Linux box that is meant to act as a router/gateway forwarding packets between two networks it's attached to — e.g. `0` = disabled, `1` = enabled):
  ```
  $ cat /proc/sys/net/ipv4/ip_forward
  0

  $ echo 1 > /proc/sys/net/ipv4/ip_forward
  ```
  - **Exam-relevant:** `echo 1 > /proc/sys/net/ipv4/ip_forward` enables IP forwarding **immediately but only until reboot** — it's a live, in-memory kernel setting change, not a persistent one.
- **To enable packet forwarding for IPv4 persistently**, edit `/etc/sysctl.conf`:
  ```
  $ cat /etc/sysctl.conf

  # Uncomment the line
  net.ipv4.ip_forward=1
  ```
- **To view the sysctl variables:**
  ```
  $ sysctl -a
  ```
- **To reload the sysctl configuration** (apply changes made in `/etc/sysctl.conf` without rebooting):
  ```
  $ sysctl --system
  ```
- **Exam-relevant summary for this file:** `ip link`/`ip addr` inspect Layer-2/interface info (switching); `route`/`ip route show`/`ip route list` inspect the routing table, `ip route add <subnet> via <gateway>` adds a specific route and `ip route add default via <gateway>` adds the catch-all default route (Layer 3/gateway); and `/proc/sys/net/ipv4/ip_forward` (live) plus `/etc/sysctl.conf`'s `net.ipv4.ip_forward=1` (persistent, applied via `sysctl --system`) control whether a Linux host will actually forward packets between interfaces — a prerequisite for that host to function as a router/gateway. This exact IP-forwarding mechanism reappears later when Linux hosts act as gateways for network namespaces and Docker bridge networks (files 05–06).

---

## 03. Pre-requisite — DNS

- Video/lecture reference: https://kodekloud.com/topic/prerequsite-dns/
- Topic: DNS (Domain Name System) fundamentals on Linux — how hostnames get resolved to IP addresses, the relevant configuration files, and the command-line tools used to test/troubleshoot resolution.

### Name Resolution

- **Checking reachability of an IP address on the network** with `ping`:
  ```
  $ ping 172.17.0.64
  PING 172.17.0.64 (172.17.0.64) 56(84) bytes of data.
  64 bytes from 172.17.0.64: icmp_seq=1 ttl=64 time=0.384 ms
  64 bytes from 172.17.0.64: icmp_seq=2 ttl=64 time=0.415 ms
  ```
- **Checking with a hostname instead of an IP** — fails, because nothing yet tells the host what IP `web` maps to:
  ```
  $ ping web
  ping: unknown host web
  ```
- **Adding an entry to the `/etc/hosts` file** to resolve the hostname locally:
  ```
  $ cat >> /etc/hosts
  172.17.0.64  web


  # Ctrl + c to exit
  ```
- **Now name resolution works, because the system looks into the `/etc/hosts` file:**
  ```
  $ ping web
  PING web (172.17.0.64) 56(84) bytes of data.
  64 bytes from web (172.17.0.64): icmp_seq=1 ttl=64 time=0.491 ms
  64 bytes from web (172.17.0.64): icmp_seq=2 ttl=64 time=0.636 ms

  $ ssh web

  $ curl http://web
  ```
  - **Exam-relevant:** `/etc/hosts` is checked for *any* tool that resolves names through the standard C library resolver — not just `ping`, but `ssh`, `curl`, etc. all benefit from the same static hostname mapping.

### DNS

- **Every host has a DNS resolution configuration file at `/etc/resolv.conf`** — this specifies which DNS server(s) (nameservers) the host should query when a name isn't found in `/etc/hosts`:
  ```
  $ cat /etc/resolv.conf
  nameserver 127.0.0.53
  options edns0
  ```
- **To change the order in which name-resolution sources are consulted** (e.g. whether local `/etc/hosts` is checked before or after DNS), edit `/etc/nsswitch.conf`:
  ```
  $ cat /etc/nsswitch.conf

  hosts:          files dns
  networks:       files
  ```
  - `hosts: files dns` means: check the `files` source (`/etc/hosts`) **first**, then fall back to `dns` (the nameserver(s) in `/etc/resolv.conf`) if not found there.
- **If resolution fails** (e.g. the nameserver configured can't resolve external names):
  ```
  $ ping wwww.github.com
  ping: www.github.com: Temporary failure in name resolution
  ```
- **Fix: add a well-known public nameserver to `/etc/resolv.conf`:**
  ```
  $ cat /etc/resolv.conf
  nameserver   127.0.0.53
  nameserver   8.8.8.8
  options edns0
  ```
  ```
  $ ping www.github.com
  PING github.com (140.82.121.3) 56(84) bytes of data.
  64 bytes from 140.82.121.3 (140.82.121.3): icmp_seq=1 ttl=57 time=7.07 ms
  64 bytes from 140.82.121.3 (140.82.121.3): icmp_seq=2 ttl=57 time=5.42 ms
  ```
  - `8.8.8.8` is Google's well-known public DNS server, added here as a second/fallback `nameserver` line so the host can resolve names when the primary (`127.0.0.53`, a local stub resolver) isn't sufficient.

### Domain Names

- **Inferred context (image `net8.PNG`):** given the section heading and its position right after the `/etc/resolv.conf`/public-nameserver discussion, this most likely illustrates the **hierarchical structure of domain names** — e.g. breaking down a fully-qualified domain name like `web.mycompany.com` into its constituent parts (host label `web`, second-level domain `mycompany`, top-level domain `com`), and/or the general DNS hierarchy from the root (`.`) down through TLDs (`.com`, `.org`, etc.) to registered domains and their subdomains/hostnames — setting up the vocabulary (`domain`, `subdomain`, `FQDN`) that the following "Search Domain" section builds on.

### Search Domain

- **Inferred context (image `net9.PNG`):** likely illustrates the **search domain / search list** concept — a `search` directive in `/etc/resolv.conf` (e.g. `search mycompany.com`) that lets a user type a short, unqualified hostname (e.g. `web`) and have the resolver automatically try appending the configured suffix (`web.mycompany.com`) before giving up, so that internal/company hosts can be reached without typing the fully-qualified name every time — directly foreshadowing how Kubernetes' own CoreDNS configures Pod `/etc/resolv.conf` files with a `search` list of `<namespace>.svc.cluster.local`, `svc.cluster.local`, `cluster.local`, etc. so Pods can resolve Services by short name.

### Record Types

- **Inferred context (image `net10.PNG`):** likely a table of common **DNS record types**, most plausibly including:
  - **A** record — maps a hostname to an **IPv4** address.
  - **AAAA** record — maps a hostname to an **IPv6** address.
  - **CNAME** record — an alias mapping one hostname to another hostname (canonical name).
  - Possibly also **MX** (mail exchange), **TXT**, **NS**, **PTR** (reverse lookup) as commonly-taught examples in an introductory DNS-record-types slide.
  - This is exactly the vocabulary (`A` vs `CNAME` records) reused later in the course when discussing how Kubernetes Services get DNS entries via CoreDNS.

### Networking Tools

- Useful networking tools to test DNS name resolution:

#### nslookup

```
$ nslookup www.google.com
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   www.google.com
Address: 172.217.18.4
Name:   www.google.com
```

#### dig

```
$ dig www.google.com

; <<>> DiG 9.11.3-1 ...
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8738
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;www.google.com.                        IN      A

;; ANSWER SECTION:
www.google.com.         63      IN      A       216.58.206.4

;; Query time: 6 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
```

- **Exam-relevant callout:** `nslookup` and `dig` both query DNS directly (bypassing/ignoring `/etc/hosts`'s `files` source entirely — unlike `ping`), making them the go-to tools specifically for **diagnosing DNS server behavior** itself, as opposed to overall name-resolution behavior. This distinction becomes important later in the course when troubleshooting CoreDNS in a Kubernetes cluster — `nslookup`/`dig` run from inside a Pod tell you what CoreDNS is actually returning, independent of any local hosts-file entries.

---

## 04. Pre-requisite — CoreDNS

- Video/lecture reference: https://kodekloud.com/topic/prerequisite-coredns/
- Topic: a hands-on introduction to running **CoreDNS** as a standalone DNS server — the exact same DNS server software Kubernetes uses internally for cluster DNS (covered later in the course, files 19–21 of the source section).

### Installation of CoreDNS

```
$ wget https://github.com/coredns/coredns/releases/download/v1.7.0/coredns_1.7.0_linux_amd64.tgz
coredns_1.7.0_linux_amd64.tgz
```

### Extract tar file

```
$ tar -xzvf coredns_1.7.0_linux_amd64.tgz
coredns
```

### Run the executable file

- Run the executable file to start a DNS server. By default, it listens on **port 53**, which is the default port for a DNS server.
```
$ ./coredns
```

### Configuring the hosts file

- Adding entries into the `/etc/hosts` file.
- **CoreDNS will pick up the IPs and names from the `/etc/hosts` file on the server** (when configured to do so via the `hosts` plugin — see the Corefile below).
```
$ cat > /etc/hosts
192.168.1.10    web
192.168.1.11    db
192.168.1.15    web-1
192.168.1.16    db-1
192.168.1.21    web-2
192.168.1.22    db-2
```

### Adding into the Corefile

- CoreDNS is configured via a file named **`Corefile`**, using a plugin-chain syntax. Here, the `hosts` plugin is pointed at `/etc/hosts` so that CoreDNS serves DNS answers sourced from that file:
```
$ cat > Corefile
. {
	hosts   /etc/hosts
}
```
- `.` is the DNS zone this block applies to (`.` = the root zone, i.e. "all queries").
- `hosts /etc/hosts` is a CoreDNS **plugin** directive telling CoreDNS to resolve names using the entries in `/etc/hosts`, effectively turning that static file into a live DNS server's answers.

### Run the executable file (again)

- Re-run CoreDNS so it picks up the new `Corefile` and starts serving DNS answers derived from `/etc/hosts`:
```
$ ./coredns
```
- **Exam-relevant takeaway:** this simple `hosts`-plugin `Corefile` is conceptually identical to how the Kubernetes-flavored CoreDNS deployment works later in the course — except in-cluster CoreDNS uses the **`kubernetes` plugin** instead of (or alongside) `hosts`, which dynamically sources DNS records from the cluster's own Service/Pod objects (via the API server) instead of a static file. Understanding this `Corefile`/plugin-chain concept here is the direct prerequisite for understanding the in-cluster CoreDNS `Corefile` (ConfigMap) covered in file 20 of the source section.

#### Reference Docs

- https://github.com/kubernetes/dns/blob/master/docs/specification.md
- https://coredns.io/plugins/kubernetes/
- https://github.com/coredns/coredns/releases

---

## 05. Pre-requisite — Network Namespace

- Video/lecture reference: https://kodekloud.com/topic/prerequsite-network-namespaces/
- Topic: Linux network namespaces — the kernel isolation primitive that gives each container (and, later, each Kubernetes Pod) its own private view of the network stack (interfaces, routing table, ARP table, iptables rules) — plus how to connect isolated namespaces back together using virtual Ethernet (veth) pairs and Linux bridges.

### Process Namespace

- On the **container**:
  ```
  $ ps aux
  ```
- On the **host**:
  ```
  $ ps aux
  ```
  - The point of this comparison (matching the general Docker/container-isolation teaching pattern this course uses elsewhere): a container has its **own PID namespace**, so `ps aux` run *inside* a container shows only that container's own processes (e.g. PID 1 being the container's main process), whereas `ps aux` run on the **host** shows *all* processes across the whole system, including the container's processes visible under their real host-side PIDs — establishing the general concept of Linux namespaces (process isolation) before narrowing specifically to **network** namespaces.

### Network Namespace

- Every network namespace gets its **own** routing table and ARP table, isolated from the host's and from every other namespace's:
  ```
  $ route
  ```
  ```
  $ arp
  ```

### Create Network Namespace

- **Create two network namespaces**, named `red` and `blue`:
  ```
  $ ip netns add red

  $ ip netns add blue
  ```
- **List the network namespaces:**
  ```
  $ ip netns
  ```

### Exec in Network Namespace

- **List the interfaces on the host:**
  ```
  $ ip link
  ```
- **Exec inside a network namespace** to see *its* interfaces (each new namespace starts with just an isolated, `DOWN` loopback interface — no connectivity to anything):
  ```
  $ ip netns exec red ip link
  1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

  $ ip netns exec blue ip link
  1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  ```
- **Alternate syntax** — `ip -n <namespace> <command>` works identically to `ip netns exec <namespace> <command>`:
  ```
  $ ip -n red link
  1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  ```

### ARP and Routing Table

- **On the host** — the host's ARP table shows real neighbor MAC/IP mappings learned from its own interface(s) (e.g. `ens3`):
  ```
  $ arp
  Address                  HWtype  HWaddress           Flags Mask            Iface
  172.17.0.21              ether   02:42:ac:11:00:15   C                     ens3
  172.17.0.55              ether   02:42:ac:11:00:37   C                     ens3
  ```
- **On the network namespaces** — completely empty, because `red`/`blue` have no interfaces (other than an isolated, down loopback) and therefore no neighbors to have learned about at all:
  ```
  $ ip netns exec red arp
  Address                  HWtype  HWaddress           Flags Mask            Iface

  $ ip netns exec blue arp
  Address                  HWtype  HWaddress           Flags Mask            Iface
  ```
- **On the host:**
  ```
  $ route
  ```
- **On the network namespaces** — again empty, no routing table entries at all yet:
  ```
  $ ip netns exec red route
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface

  $ ip netns exec blue route
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  ```
- **Exam-relevant:** this demonstrates the core isolation property of network namespaces — `red` and `blue` are each a completely separate network stack, with **no** shared interfaces, ARP entries, or routes with the host or each other, until you explicitly wire them together (next section).

### Virtual Cable (veth pair)

- **To create a virtual cable** — a **veth (virtual Ethernet) pair** acts like a virtual patch cable with two ends; anything sent into one end comes out the other:
  ```
  $ ip link add veth-red type veth peer name veth-blue
  ```
- **To attach each end to a different network namespace:**
  ```
  $ ip link set veth-red netns red

  $ ip link set veth-blue netns blue
  ```
- **To add an IP address to each end:**
  ```
  $ ip -n red addr add 192.168.15.1/24 dev veth-red

  $ ip -n blue addr add 192.168.15.2/24 dev veth-blue
  ```
- **To bring the namespace-side interfaces up:**
  ```
  $ ip -n red link set veth-red up

  $ ip -n blue link set veth-blue up
  ```
- **Check reachability** — pinging across the veth pair from `red` to `blue`, and inspecting the resulting ARP entries showing the veth interfaces' MAC addresses now populated on each side:
  ```
  $ ip netns exec red ping 192.168.15.2
  PING 192.168.15.2 (192.168.15.2) 56(84) bytes of data.
  64 bytes from 192.168.15.2: icmp_seq=1 ttl=64 time=0.035 ms
  64 bytes from 192.168.15.2: icmp_seq=2 ttl=64 time=0.046 ms

  $ ip netns exec red arp
  Address                  HWtype  HWaddress           Flags Mask            Iface
  192.168.15.2             ether   da:a7:29:c4:5a:45   C                     veth-red

  $ ip netns exec blue arp
  Address                  HWtype  HWaddress           Flags Mask            Iface
  192.168.15.1             ether   92:d1:52:38:c8:bc   C                     veth-blue
  ```
- **Delete the link** (deleting either end of a veth pair removes both ends of the virtual cable entirely):
  ```
  $ ip -n red link del veth-red
  ```
- **On the host after deletion** — the veth interfaces never showed up in the host's own ARP table anyway (they only exist inside the `red`/`blue` namespaces), so the host's `arp` output only ever showed unrelated, pre-existing entries on its real interface (`ens3`):
  ```
  # Not available
  $ arp
  Address                  HWtype  HWaddress           Flags Mask            Iface
  172.16.0.72              ether   06:fe:61:1a:75:47   C                     ens3
  172.17.0.68              ether   02:42:ac:11:00:44   C                     ens3
  172.17.0.74              ether   02:42:ac:11:00:4a   C                     ens3
  172.17.0.75              ether   02:42:ac:11:00:4b   C                     ens3
  ```
  - **Note:** a veth pair directly connecting exactly two namespaces **does not scale** — it only works for point-to-point connectivity between two specific namespaces. Connecting many namespaces (e.g. many containers/Pods on one host) to each other requires a **switch-like** device instead — which is exactly the motivation for the Linux Bridge covered next.

### Linux Bridge

- **Create network namespaces** (same as before):
  ```
  $ ip netns add red

  $ ip netns add blue
  ```
- **To create an internal virtual bridge network**, add a new interface of type `bridge` to the host — this acts like a virtual switch that any number of namespaces/veth pairs can be connected to:
  ```
  $ ip link add v-net-0 type bridge
  ```
- **Display it on the host** (note it's created in a `DOWN` state initially):
  ```
  $ ip link
  8: v-net-0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/ether fa:fd:d4:9b:33:66 brd ff:ff:ff:ff:ff:ff
  ```
- **Bring it up:**
  ```
  $ ip link set dev v-net-0 up
  ```
- **To connect each network namespace to the bridge**, create a veth pair per namespace — one end will stay in the namespace, the other end will attach to the bridge:
  ```
  $ ip link add veth-red type veth peer name veth-red-br

  $ ip link add veth-blue type veth peer name veth-blue-br
  ```
- **Move one end of each pair into its namespace, and attach the other end to the bridge** (`master v-net-0` makes that interface a bridge port, i.e. plugs the virtual cable into the virtual switch):
  ```
  $ ip link set veth-red netns red

  $ ip link set veth-blue netns blue

  $ ip link set veth-red-br master v-net-0

  $ ip link set veth-blue-br master v-net-0
  ```
- **Add an IP address to each namespace-side interface:**
  ```
  $ ip -n red addr add 192.168.15.1/24 dev veth-red

  $ ip -n blue addr add 192.168.15.2/24 dev veth-blue
  ```
- **Bring the namespace-side interfaces up:**
  ```
  $ ip -n red link set veth-red up

  $ ip -n blue link set veth-blue up
  ```
- **Add an IP address to the bridge itself on the host**, so the host has an address on this virtual network too (this becomes the "gateway" address for `red`/`blue`, mirroring the Gateway concept from file 02):
  ```
  $ ip addr add 192.168.15.5/24 dev v-net-0
  ```
- **Bring the host-side (bridge-port) interfaces up:**
  ```
  $ ip link set dev veth-red-br up
  $ ip link set dev veth-blue-br up
  ```
- **Inferred context (implied diagram, based on this walkthrough):** the resulting topology is a Linux bridge `v-net-0` (IP `192.168.15.5/24`) acting as a virtual switch on the host, with two veth pairs plugged into it — `veth-red`↔`veth-red-br` connecting namespace `red` (`192.168.15.1/24`) to the bridge, and `veth-blue`↔`veth-blue-br` connecting namespace `blue` (`192.168.15.2/24`) to the bridge — exactly the same fan-out pattern Docker later uses for its own `docker0` bridge (file 06) and that CNI bridge plugins use for Kubernetes Pods.
- **From the host, the bridge network is directly reachable** (same subnet as the bridge's own IP):
  ```
  $ ping 192.168.15.1
  ```
- **From inside a namespace, reaching anything *outside* the bridge's own subnet fails** until a route is added — because `blue` has no routing table entry pointing anywhere except its own directly-connected subnet:
  ```
  $ ip netns exec blue ping 192.168.1.1
  Connect: Network is unreachable

  $ ip netns exec blue route

  $ ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5

  # Check the IP Address of the host
  $ ip a

  $ ip netns exec blue ping 192.168.1.1
  PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
  ```
  - Adding the route `192.168.1.0/24 via 192.168.15.5` tells namespace `blue` to send anything destined for `192.168.1.0/24` via the bridge's host-side IP (`192.168.15.5`), which then relies on the **host's own routing table** to actually get it from there onward (i.e. the host acts as a router for the namespace, exactly mirroring the Gateway concept from file 02).
- **Reaching the internet (`8.8.8.8`) additionally requires NAT (masquerading)**, because `192.168.15.0/24` is a private, internal-only subnet that upstream routers/the internet don't know how to route back to — the host must rewrite the namespace's private source IP to its own public/routable IP on the way out:
  ```
  $ iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE

  $ ip netns exec blue ping 192.168.1.1

  $ ip netns exec blue ping 8.8.8.8

  $ ip netns exec blue route

  $ ip netns exec blue ip route add default via 192.168.15.5

  $ ip netns exec blue ping 8.8.8.8
  ```
  - `iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE` — appends a rule to the **`nat`** table's **`POSTROUTING`** chain that source-NATs (masquerades) any outgoing packet whose source is in `192.168.15.0/24`, replacing it with the host's own outbound-interface IP — this is exactly the same mechanism Docker uses for its bridge network's outbound internet access (file 06), and the same general mechanism behind Kubernetes Pod-to-external-internet traffic.
  - After adding a **default route** (`ip netns exec blue ip route add default via 192.168.15.5`) inside `blue`, *any* destination not otherwise matched (not just `192.168.1.0/24`) is sent via the bridge/host — so `8.8.8.8` (a public internet address) now successfully routes out through the host and gets masqueraded.
- **Adding a port-forwarding rule to iptables** — DNAT (Destination NAT) rewrites the destination of incoming packets, letting an external client reach a service running inside a namespace by hitting the host's own IP/port instead:
  ```
  $ iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.15.2:80 -j DNAT
  ```
  - This appends a rule to the **`PREROUTING`** chain of the **`nat`** table: any incoming packet destined for port `80` gets its destination rewritten to `192.168.15.2:80` (namespace `blue`'s IP) — this exact DNAT pattern is what Docker's own published-port (`-p hostPort:containerPort`) feature is implemented with under the hood (see file 06).
- **List the iptables NAT rules** (to verify the MASQUERADE/DNAT rules above actually got added, and inspect packet/byte counters):
  ```
  $ iptables -nvL -t nat
  ```
- **Exam-relevant summary for this file:** `ip netns add/list/exec` manage namespaces; a **veth pair** (`ip link add <a> type veth peer name <b>`) connects exactly two namespaces point-to-point; a **Linux bridge** (`ip link add <name> type bridge`, `ip link set <iface> master <bridge>`) acts as a virtual switch letting many namespaces/veth pairs connect together and to the host, which can then act as a **router/gateway** for those namespaces (needing routes added inside each namespace, plus the host's own `ip_forward` sysctl enabled per file 02); and reaching networks/the internet beyond the bridge's own subnet requires **iptables NAT** — `MASQUERADE` in `POSTROUTING` for outbound source-NAT, and `DNAT` in `PREROUTING` for inbound port-forwarding. This entire chapter is a hands-on, manual walkthrough of **exactly** what Docker automates for you with `docker0` (file 06) and what CNI plugins automate for you in Kubernetes.

---

## 06. Pre-requisite — Docker Networking

- Video/lecture reference: https://kodekloud.com/topic/prerequsite-docker-networking/
- Topic: how Docker implements container networking using the exact primitives from file 05 (namespaces, veth pairs, a Linux bridge, iptables NAT) — packaged into three selectable networking modes (`none`, `host`, `bridge`).

### None Network

- Running a Docker container with the `none` network — the container gets **no network interfaces at all** (other than loopback), i.e. it is completely network-isolated:
  ```
  $ docker run --network none nginx
  ```

### Host Network

- Running a Docker container with the `host` network — the container **shares the host's own network namespace** entirely (no isolation, no separate IP; the container sees/uses the host's interfaces directly, and anything it listens on binds directly to the host's ports):
  ```
  $ docker run --network host nginx
  ```

### Bridge Network

- Running a Docker container with the `bridge` network — the **default** mode: the container gets its own private network namespace, connected to the host via a veth pair plumbed into Docker's own Linux bridge (`docker0`), exactly matching the manual Linux Bridge walkthrough in file 05:
  ```
  $ docker run --network bridge nginx
  ```

### List the Docker Networks

```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
4974cba36c8e        bridge              bridge              local
0e7b30a6c996        host                host                local
a4b19b17d2c5        none                null                local
```

### To view the Network Device on the Host

```
$ ip link
or
$ ip link show docker0
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:cf:c3:df:f5 brd ff:ff:ff:ff:ff:ff
```

- **With the `ip link add` command, you could set up a bridge device named `docker0` yourself** (illustrating that `docker0` is nothing special/magic — it's a plain Linux bridge, created the same way file 05 created `v-net-0`; Docker just does this automatically for you):
  ```
  $ ip link add docker0 type bridge
  ```

### To view the IP Address of the interface `docker0`

```
$ ip addr
or
$ ip addr show docker0
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:cf:c3:df:f5 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/24 brd 172.18.0.255 scope global docker0
       valid_lft forever preferred_lft forever
```

- **Exam-relevant:** `docker0`'s own address (`172.18.0.1/24` here) is the gateway address every bridge-networked container on this host will use — the same "bridge has an IP, acts as gateway" pattern from file 05's `v-net-0` (`192.168.15.5/24`).

### Run the command to create a Docker Container

```
$ docker run nginx
```

### To list the Network Namespaces

```
$ ip netns
1c452d473e2a (id: 2)
db732004aa9b (id: 1)
04acb487a641 (id: 0)
default
```

- **Inspect the Docker container:**
  ```
  $ docker inspect <container-id>
  ```
- **To view the interface attached to the local bridge `docker0`** — the host-side end of the container's veth pair shows up as a `veth...@if<N>` interface with `master docker0` (i.e. it's a bridge port on `docker0`, exactly like `veth-red-br`/`veth-blue-br` were bridge ports on `v-net-0` in file 05):
  ```
  $ ip link
  3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
      link/ether 02:42:c8:3a:ea:67 brd ff:ff:ff:ff:ff:ff
  5: vetha3e33331@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
      link/ether e2:b2:ad:c9:8b:98 brd ff:ff:ff:ff:ff:ff link-netnsid 0
  ```
- **Using the `-n` option with the network namespace ID to view the other end of the veth pair** — i.e. what that same interface looks like *inside* the container's own network namespace (it shows up there as `eth0`, the container's normal-looking primary interface):
  ```
  $ ip -n 04acb487a641 link
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  3: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
      link/ether c6:f3:ca:12:5e:74 brd ff:ff:ff:ff:ff:ff link-netnsid 0
  ```
- **To view the IP address assigned to this interface** (the container's own private, `docker0`-subnet IP):
  ```
  $ ip -n 04acb487a641 addr
  3: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
      link/ether c6:f3:ca:12:5e:74 brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 10.244.0.2/24 scope global eth0
         valid_lft forever preferred_lft forever
  ```
- **Exam-relevant recap:** every Docker container on the `bridge` network is, under the hood, just another Linux network namespace connected to the `docker0` bridge via a veth pair — `eth0@if5` inside the container's namespace is the **same virtual cable** as `vetha3e33331@if3` on the host side, precisely the veth-pair-plus-bridge pattern manually built by hand in file 05.

### Port Mapping

- **Creating a Docker container:**
  ```
  $ docker run -itd --name nginx nginx
  d74ca9d57c1d8983db2c590df2fdd109e07e1972d6b361a6ecad8a942af5bf7e
  ```
- **Inspect the Docker container to view its IP address:**
  ```
  $ docker inspect nginx | grep -w IPAddress
              "IPAddress": "172.18.0.6",
                      "IPAddress": "172.18.0.6",
  ```
- **Accessing the web page with `curl`** — works fine **from the host** directly to the container's private bridge IP:
  ```
  $ curl --head  http://172.18.0.6:80
  HTTP/1.1 200 OK
  Server: nginx/1.19.2
  ```
- **Problem:** the container's `172.18.0.6` IP is only reachable from the Docker host itself (it's on a private, internal-only bridge subnet) — an external client can't reach it directly. **Port Mapping** solves this by publishing a port on the host itself that forwards into the container:
  ```
  $ docker run -itd --name nginx -p 8080:80 nginx
  e7387bbb2e2b6cc1d2096a080445a6b83f2faeb30be74c41741fe7891402f6b6
  ```
  - `-p 8080:80` maps **host port `8080`** to **container port `80`**.
- **Inspecting the Docker container to view the assigned ports:**
  ```
  $ docker inspect nginx | grep -w -A5 Ports

    "Ports": {
                  "80/tcp": [
                      {
                          "HostIp": "0.0.0.0",
                          "HostPort": "8080"
                      }

  ```
- **To view the IP address of the host system, and access nginx externally via the host's IP and the mapped port:**
  ```
  $ ip a

  # Accessing nginx page with curl command

  $ curl --head http://192.168.10.11:8080
  HTTP/1.1 200 OK
  Server: nginx/1.19.2
  ```
- **Configuring the underlying `iptables nat` rules** that Docker's `-p 8080:80` flag actually creates for you automatically — a two-hop DNAT (first redirecting the host's own published port, then routing it on to the container's actual bridge IP:port), exactly mirroring the manual `PREROUTING`/`DNAT` rule built by hand at the end of file 05:
  ```
  $ iptables \
           -t nat \
           -A PREROUTING \
           -j DNAT \
           --dport 8080 \
           --to-destination 80
  ```
  ```
  $ iptables \
        -t nat \
        -A DOCKER \
        -j DNAT \
        --dport 8080 \
        --to-destination 172.18.0.6:80
  ```

### List the Iptables rules

```
$ iptables -nvL -t nat
```

- **Exam-relevant summary for this file:** Docker offers three network modes — **`none`** (fully isolated, no networking), **`host`** (shares the host's network namespace directly, no isolation), and **`bridge`** (the default — private namespace per container, connected via veth pair to the `docker0` Linux bridge, same mechanism as file 05's hand-built `v-net-0`). `docker network ls` lists them; `ip link`/`ip addr [show docker0]` inspect the bridge on the host; `ip -n <container-netns-id> link/addr` inspects the container's own side of its veth pair (`docker inspect` reveals the container's namespace ID/IP). **Port mapping** (`docker run -p <hostPort>:<containerPort>`) is implemented purely via **iptables DNAT rules** in the `nat` table, letting external traffic reach a container's private bridge IP through a published host port.

#### Reference docs

- https://docs.docker.com/network/
- https://linux.die.net/man/8/iptables
- https://linux.die.net/man/8/ip

---

## 07. Pre-requisite — CNI (Container Network Interface)

- Video/lecture reference: https://kodekloud.com/topic/prerequsite-cni/
- Topic: the **Container Network Interface (CNI)** — a standardized specification/contract that lets container runtimes (Docker, containerd, rkt, Kubernetes' kubelet, etc.) delegate the actual work of "set up networking for this container" to a pluggable, swappable third-party plugin, rather than every runtime having to hand-roll its own bridge/veth/iptables logic (as walked through manually in files 05–06).

### The CNI problem/contract

- **Inferred context (image `net7.PNG`):** given this is the only diagram in the file and it appears immediately under the page title (before any of the concrete plugin content), this is almost certainly the canonical **CNI invocation-contract diagram** — showing a **container runtime** (e.g. the kubelet, or a generic "runtime" box) on one side that, whenever a container is created or deleted, invokes an external **CNI plugin executable** (e.g. `bridge`) with a standardized interface: the runtime passes the plugin a **network namespace** to configure, a **container ID**, and a JSON **network configuration**, and calls the plugin executable with a specific verb (`ADD` to set up networking, `DEL` to tear it down) via environment variables/stdin; the plugin then does the actual low-level work (creating a veth pair, attaching one end to a bridge, assigning an IP, setting routes — exactly the file-05/06 primitives) and returns the resulting interface/IP info as JSON on stdout. This diagram is the conceptual bridge between "manually running `ip link`/`ip netns`/`iptables` commands by hand" (files 05–06) and "letting a pluggable CNI binary do it for you" — which is exactly how Kubernetes wires up Pod networking later in the course (see file 11, "CNI in Kubernetes").
- **Exam-relevant callout (general CNI knowledge underlying this lesson):** the CNI spec defines that plugins are simple **executables** placed in a well-known directory, invoked directly by the runtime — there's no long-running daemon required by the spec itself (though many real-world plugins, like Weave or Calico, also run their own daemons/agents alongside the CNI binary for additional functionality like cross-node routing).

### Third-Party Network Plugin Providers

- [Weave](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-installation)
- [Calico](https://docs.projectcalico.org/getting-started/kubernetes/quickstart)
- [Flannel](https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md)
- [Cilium](https://github.com/cilium/cilium)
- These are all **CNI-compliant plugins** — any one of them can be "plugged in" to a Kubernetes cluster (or any other CNI-compliant runtime) interchangeably, because they all speak the same standardized CNI contract described above, even though their internal networking approaches differ significantly (e.g. overlay networks vs. BGP-based routing vs. eBPF).

### To view the CNI Network Plugins

- CNI ships with a set of supported/reference network plugins, installed as executables in a standard directory:
  ```
  $ ls /opt/cni/bin/
  bridge  dhcp  flannel  host-device  host-local  ipvlan  loopback  macvlan  portmap  ptp  sample  tuning  vlan
  ```
  - **Exam-relevant:** `/opt/cni/bin/` is the conventional filesystem location where CNI plugin binaries live on a node — this exact path (and the presence of a `bridge` plugin binary in particular) reappears later in the course when configuring/troubleshooting Kubernetes' own CNI setup (file 11, "CNI in Kubernetes"), since the kubelet is configured to look in this directory (via `--cni-bin-dir`) for the plugin binary named in the active CNI network configuration file.
  - Notable binaries in this list: **`bridge`** (implements the exact veth-pair + Linux-bridge pattern from files 05–06, but automated/standardized via the CNI contract), **`loopback`** (sets up the container's `lo` interface), **`host-local`** (an IP Address Management/IPAM plugin that allocates IPs from a local, statically-configured range), **`dhcp`** (an IPAM plugin that requests IPs from a DHCP server instead), **`macvlan`**/**`ipvlan`**/**`vlan`** (alternative Layer-2/Layer-3 virtual-interface plugin types), and **`portmap`** (implements port-mapping/publishing, conceptually equivalent to Docker's `-p hostPort:containerPort` iptables DNAT trick from file 06).

#### Reference Docs

- https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/

---

*Continued in [kubernetes-networking-notes_part_02.md](kubernetes-networking-notes_part_02.md)
— Sections 08–14 (Cluster Networking, Practice Test — Explore Env, Pod Networking, CNI
in Kubernetes, CNI Weave, Practice Test — CNI Weave, Practice Test — Deploy Network
Solution).*
