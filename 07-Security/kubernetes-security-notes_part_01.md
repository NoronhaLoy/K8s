# Security — Complete Notes (Part 1 of 4)

> **Part 1 of 4** — covers Sections 01–08 (Section Introduction, Kubernetes Security
> Primitives, Authentication, TLS Certificates Introduction, TLS Basics, TLS in
> Kubernetes, TLS in Kubernetes — Certificate Creation, View Certificate Details).
> Next: [kubernetes-security-notes_part_02.md](kubernetes-security-notes_part_02.md)
> (Sections 09–15).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/07-Security/`

---

## 01. Security — Section Introduction

- Video reference: *Security Section Introduction* (KodeKloud) — https://kodekloud.com/topic/security-section-introduction/
- This section of the CKA course covers the full Kubernetes security curriculum, introduced as a list of upcoming topics:
  - **Kubernetes Security Primitives**
  - **Authentication**
  - **TLS certificates for cluster components**
  - **Secure Persistent key-value store** (i.e. securing etcd)
  - **Authorization**
  - **Images Security**
  - **Security Contexts**
  - **Network Policies**
- Big picture: this is the largest, most heavily-weighted section of the CKA exam curriculum — it spans everything from who can authenticate to the cluster, to how the cluster's internal components trust each other via TLS, to what an authenticated identity is *allowed* to do (RBAC/authorization), to workload-level hardening (image security, security contexts, network policies). Sections 01–15 cover the **first half** of the Security section: primitives, authentication, TLS/PKI, the Kubernetes Certificates API, kubeconfig, and API groups. (RBAC, Cluster Roles, Service Accounts, Image Security, Security Contexts, and Network Policies follow in the second half, numbered 16–30 in the source docs.)

---

## 02. Kubernetes Security Primitives

- Video reference: https://kodekloud.com/topic/kubernetes-security-primitives/
- Topic: the foundational security concepts that apply to *any* host running services, then how those concepts map onto a Kubernetes cluster specifically.

### Secure Hosts

- **Inferred context (image `sech.PNG`):** likely a generic "securing a host" checklist diagram — the kind of baseline OS/host-hardening steps that apply to any server before you even get to Kubernetes-specific security, such as: disabling root/password SSH login in favor of key-based access, disabling unnecessary services/ports, keeping the OS patched, and restricting who has shell access to the underlying node. The point of this slide is that Kubernetes security is built **on top of** — and depends on — the underlying hosts already being secured; a compromised node undermines every other Kubernetes-level protection.

### Secure Kubernetes

- Securing a Kubernetes cluster requires making **two types of decisions**:
  - **Who can access [the cluster]?** — this is the **Authentication** question.
  - **What can they do?** — this is the **Authorization** question.
- **Inferred context (image `seck.PNG`):** likely a two-box diagram — a person/robot icon on the left approaching a gate labeled "Authentication" (Who can access?), and once through the gate, a second gate labeled "Authorization" (What can they do?) guarding access to cluster resources — visually splitting cluster security into these two sequential concerns, which the rest of the Security section is structured around (Authentication in files 03–14; Authorization/RBAC in the later files of the source section).

### Authentication

- **Who can access the API Server** is defined by the **Authentication** mechanisms.
- **Exam-relevant:** the `kube-apiserver` is the single front door to the cluster — every authentication decision is ultimately about controlling requests hitting the API server.

### Authorization

- Once a user/process gains access to the cluster, **what they can do** is defined by **Authorization** mechanisms (e.g. RBAC, ABAC, Node Authorization, Webhook).

### TLS Certificates

- **All communication with the cluster, between the various components** — such as the **ETCD Cluster**, **kube-controller-manager**, **scheduler**, **api server**, as well as those running on the worker nodes such as the **kubelet** and **kube-proxy** — **is secured using TLS encryption.**
- **Inferred context (image `tls.PNG`):** likely a cluster-topology diagram with every arrow between components (`etcd <-> kube-apiserver`, `kube-apiserver <-> kube-scheduler`, `kube-apiserver <-> kube-controller-manager`, `kube-apiserver <-> kubelet`, `kube-apiserver <-> kube-proxy`, `user/kubectl <-> kube-apiserver`) drawn as a padlocked/encrypted line, emphasizing that **every single line of communication in the cluster is TLS-encrypted**, not just the user-facing API. This same image is reused later in file 06 (TLS in Kubernetes) to introduce the server/client certificate split.

### Network Policies

- Raises the question: **what about communication between applications within the cluster** (as opposed to between cluster *components*)?
- **Inferred context (image `np.PNG`):** likely a simple diagram of Pods on a shared network where, by default, **all Pods can reach all other Pods** (flat, unrestricted pod network) — and then a **NetworkPolicy** object shown restricting that, e.g. only allowing traffic from a `front-end` Pod to a `back-end` Pod and blocking all other traffic — foreshadowing the dedicated Network Policies lesson later in the Security section.

---

## 03. Authentication

- Video reference: https://kodekloud.com/topic/authentication/
- Topic: who/what can authenticate against a Kubernetes cluster, and the mechanisms available to configure that.

### Accounts

- **Inferred context (image `auth1.PNG`):** likely lists the different categories of "accounts" that might try to access a cluster — end users of the deployed applications, administrators, developers, and third-party/automated services — setting up the narrowing-down that follows in the very next bullet.
- **Different users that may be accessing the cluster:** security of **end users** who access the *applications* deployed on the cluster is managed by the **applications themselves internally** — this is explicitly **out of scope** for Kubernetes' own authentication system (Kubernetes doesn't authenticate the customers hitting your web app; your app does that itself).
  - **Inferred context (image `acc1.PNG`):** likely shows an end user hitting an application (e.g. a web front-end) running *inside* the cluster, with a note/arrow clarifying that this user-to-application authentication path is handled by the app's own login system, not by Kubernetes — contrasted against administrators/developers who interact with the cluster infrastructure directly via `kubectl`/the API.
- So, after excluding application end-users, Kubernetes cluster security is left concerned with **2 types of users**:
  - **Humans** — such as Administrators and Developers.
  - **Robots** — such as other processes/services or applications that require access to the cluster (i.e. Service Accounts).
  - **Inferred context (image `acc2.PNG`):** likely a two-column diagram: a human icon (labeled "Users" — admins/developers) on one side and a robot/gear icon (labeled "Service Accounts" — processes, CI pipelines, other apps) on the other, both shown as needing to authenticate to the same API server.
- **All user access is managed by the `apiserver`, and all requests go through the `apiserver`** — regardless of whether the request originates from `kubectl`, a direct API call, or another cluster component.
  - **Inferred context (image `acc3.PNG`):** likely a hub-and-spoke diagram with `kube-apiserver` in the center and every kind of requester (human user via `kubectl`, a Service Account/Pod, another control-plane component) drawn as arrows converging on it — reinforcing that the API server is the single, universal authentication checkpoint for the entire cluster; there is no side-channel that bypasses it.

### Authentication Mechanisms

- **There are different authentication mechanisms that can be configured** on the kube-apiserver, including (per general Kubernetes knowledge underlying this exact lesson, and matching the mechanism named explicitly two headings below):
  - **Static Password File** — a CSV file mapping usernames/passwords (and optionally groups) to identities, checked via HTTP Basic Auth.
  - **Static Token File** — a CSV file mapping static bearer tokens to usernames, checked via a Bearer token in the `Authorization` header.
  - **Certificates** — client certificates (X.509), where the certificate's `CN` is the username and `O` is the group.
  - **Identity services** — external services like **LDAP**, **Kerberos**, or **SSO/OIDC** integrations.
  - **Inferred context (image `auth2.PNG`):** likely a labeled list/diagram of exactly these mechanism categories (static files, certificates, external identity services), shown as alternative pluggable modules the `kube-apiserver` can be configured to check a request's credentials against.

### Authentication Mechanisms — Basic

- **Inferred context (image `auth3.PNG`):** likely illustrates the **Basic Authentication** mechanism specifically — a simple CSV file (commonly `user-details.csv`) with rows in the form `password,username,uid[,group]`, e.g.:
  ```
  password123,user1,u0001
  password456,user2,u0002
  ```
  - This file is passed to the `kube-apiserver` so it can validate `username:password` pairs supplied on incoming requests.
  - **Exam-relevant callout (general Kubernetes knowledge):** static basic-auth/token-file authentication is a **legacy, insecure, and deprecated** mechanism (removed entirely as of Kubernetes v1.19+) — it's taught here for foundational understanding and because older exam/lab environments may still demonstrate it, but it should never be used in a real production cluster since credentials are stored in **plaintext** on disk.

### kube-apiserver configuration

- If the cluster was set up via **`kubeadm`**, you enable basic-auth by updating the **`kube-apiserver.yaml`** static Pod manifest with the appropriate option.
  - **Inferred context (image `auth4.PNG`):** likely shows the manifest snippet needed — adding a command-line flag such as:
    ```yaml
    - --basic-auth-file=/tmp/users/user-details.csv
    ```
    to the `kube-apiserver` container's `command`/`args` list inside `/etc/kubernetes/manifests/kube-apiserver.yaml`, plus mounting the directory containing the CSV file into the static Pod via a `hostPath` volume so the API server container can actually read it (since the API server runs in its own container/mount namespace).

### Authenticate User

- To authenticate using basic credentials while accessing the API server, specify the username and password in a `curl` command:
  ```
  $ curl -v -k http://master-node-ip:6443/api/v1/pods -u "user1:password123"
  ```
  - `-u "user1:password123"` supplies HTTP Basic Auth credentials.
  - `-k` skips TLS certificate verification (useful in a lab/demo where the server cert isn't trusted by the local machine).
  - **Inferred context (image `auth5.PNG`):** likely shows the corresponding sample output of this `curl` call — a JSON `PodList` response confirming the request was authenticated and authorized, i.e. proof that the basic-auth file approach works end-to-end.
- **We can have an additional column in the `user-details.csv` file to assign users to specific groups** — turning a plain `password,username,uid` row into `password,username,uid,groupname`.
  - **Inferred context (image `auth6.PNG`):** likely shows the CSV file with this 4th column populated (e.g. `password123,user1,u0001,group1`), demonstrating how group membership is derived from static auth files, which then becomes relevant when configuring RBAC RoleBindings against a **group** rather than an individual user.

### Note

- **Inferred context (image `note.PNG`):** given this immediately follows the basic-auth walkthrough and precedes the K8s reference docs link, this is almost certainly the course's explicit deprecation warning — most likely stating that **Basic Authentication (and Static Token File authentication) are insecure and have been deprecated/removed from Kubernetes** (removed in v1.19), and that these mechanisms are shown here purely for **educational purposes** to understand the underlying concept of authentication files — production clusters should rely on **certificates**, **service accounts**, or an **external identity provider (OIDC/LDAP)** instead.

### K8s Reference Docs

- https://kubernetes.io/docs/reference/access-authn-authz/authentication/

---

## 04. TLS Certificates Introduction

- Video reference: https://kodekloud.com/topic/tls-introduction/
- This is a short scene-setting file with no diagrams or commands — it simply previews the topics the following TLS lessons (files 05–14) will cover:
  - **What are TLS certificates?**
  - **How does kubernetes use certificates?**
  - **How to generate them?**
  - **How to configure them?**
  - **How to view them?**
  - **How to troubleshoot issues related to certificates?**
- **Note:** this file is purely an outline/table-of-contents page, structurally identical in purpose to other short "introduction" pages seen elsewhere in the course — no technical content of its own to extract.

---

## 05. TLS Basics

- Video reference: https://kodekloud.com/topic/tls-basics/
- Topic: foundational cryptography/TLS concepts needed before diving into how Kubernetes specifically uses certificates.

### Certificate

- **A certificate is used to guarantee trust between 2 parties during a transaction.**
- Example: when a user tries to access a web server, TLS certificates ensure that the communication between them is **encrypted**.
- **Inferred context (image `cert1.PNG`):** likely a simple browser-to-web-server diagram showing a padlock icon in the browser's address bar, representing an HTTPS connection — the visual "trust" indicator that a certificate provides to an end user, and the starting point for explaining *why* certificates matter before getting into *how* they work cryptographically.

### Symmetric Encryption

- **It is a secure way of encryption**, but it uses the **same key** to encrypt and decrypt the data, and that key has to be **exchanged** between the sender and receiver — creating a **risk of a hacker gaining access to the key** (e.g. by intercepting it in transit) and decrypting the data.
- **Inferred context (image `cert2.PNG`):** likely a diagram showing a single shared key icon used by both the sender (to lock/encrypt a message) and receiver (to unlock/decrypt it), with a "man in the middle" attacker intercepting the key during the exchange step — visually illustrating the key-distribution weakness that motivates asymmetric encryption.

### Asymmetric Encryption

- Instead of a single key, **asymmetric encryption uses a pair of keys** — a **private key** and a **public key**.
- **Inferred context (image `cert3.PNG`):** likely introduces the basic property of the key pair: data encrypted with the **public** key can only be decrypted with the corresponding **private** key (and vice-versa for signing), and the public key can be freely shared/distributed while the private key must never leave its owner's possession.
- **Inferred context (image `cert4.PNG`):** likely a concrete worked example using SSH-style key generation, e.g. `ssh-keygen`, producing a `private key` (kept secret on the local machine) and a `public key` (copied to a remote server's `authorized_keys`) — illustrating the pattern of "public key goes out into the world, private key stays home," a common teaching analogy before circling back to HTTPS/TLS.
- **Inferred context (image `cert5.PNG`):** likely shows how this pair is used for a **login/authentication** use case — a server holding the user's public key can verify a challenge signed with the matching private key, proving identity without the private key ever crossing the network (mirroring how SSH key-based login works, and foreshadowing how Kubernetes **client certificates** later serve the same authentication role).
- **Inferred context (image `cert6.PNG`):** likely pivots specifically to the **HTTPS / web server** case — the server encrypts data using its private key or negotiates a session using its key pair; the browser trusts the server's **public key** (delivered inside its certificate) to set up a secure, encrypted channel — connecting the general public/private key concept back to the "certificate guarantees trust" idea from the very first heading in this file.

### How do you look at a certificate and verify if it is legit?

- Verifying legitimacy comes down to: **who signed and issued the certificate.**
- **If you generate the certificate [yourself] then you [have] to sign it by yourself; that is known as a self-signed certificate.**
- **Inferred context (image `cert7.PNG`):** likely a browser security-warning screenshot ("Your connection is not private" / "NET::ERR_CERT_AUTHORITY_INVALID") — the classic warning a browser shows when it encounters a self-signed certificate it has no reason to trust, motivating the next question about how to get a *legitimate*, trusted certificate.

### How do you generate a legitimate certificate? How do you get your certificate signed by someone with authority?

- **That's where a `Certificate Authority (CA)` comes in for you.** Some of the popular ones are **Symantec**, **DigiCert**, **Comodo**, **GlobalSign**, etc.
- **Inferred context (image `cert8.PNG`):** likely shows the CSR (Certificate Signing Request) workflow — the applicant generates a key pair and a CSR (containing the public key + identity info), and submits the CSR to the CA rather than a self-made certificate.
- **Inferred context (image `cert9.PNG`):** likely shows the CA's side of that transaction — the CA validates the requester's identity/domain ownership, then signs the CSR using the CA's own private key, producing a certificate that is now cryptographically tied back to that trusted CA.
- **Inferred context (image `cert10.PNG`):** likely shows the payoff — a browser/OS trust store that ships with a built-in list of trusted root CA public certificates, so when it sees a server certificate signed by one of those trusted CAs, it displays the padlock/trusted-connection indicator with no warning (contrasted directly against the `cert7.PNG` self-signed warning screenshot).

### Public Key Infrastructure (PKI)

- **Inferred context (image `pki.PNG`):** likely a full end-to-end PKI diagram tying together everything covered so far in this file: a **Root CA** at the top of a trust chain, which may delegate to **Intermediate CAs**, which in turn sign **end-entity/leaf certificates** for servers and clients; each certificate carries a **public key** while its owner privately holds the matching **private key**; trust flows down the chain from the root (which browsers/OSes trust natively) to whatever leaf certificate is being presented, and this exact "root CA signs everything else" pattern is precisely how Kubernetes builds its own internal PKI in the next lesson (file 06/07) — with the cluster's own CA taking the place of a public CA like DigiCert.

### Certificate naming convention

- **Inferred context (image `cert11.PNG`):** likely summarizes the common file-extension/naming conventions used for certificates and keys across the ecosystem (this is exactly the ambiguity the course flags and resolves before moving into hands-on `openssl` commands in file 07), most plausibly covering:
  - **Public key / certificate files** are commonly named with a `.crt`, `.pem`, or `.cert` extension, and sometimes explicitly labeled e.g. `server.crt`, `client.pem`.
  - **Private key files** are commonly named with a `.key` extension, sometimes as `server.key`, `client-key.pem`, etc.
  - The takeaway (general PKI/Kubernetes convention emphasized repeatedly in the course): **whenever you see a file with `key` in the name (or a `.key` extension), treat it as a private key — keep it secret; whenever you see `.crt`/`.pem`/`cert` without "key", it's almost always a public certificate**, safe to distribute. This convention is exactly what shows up moments later in file 07's `openssl` commands (`ca.key` vs `ca.crt`, `admin.key` vs `admin.crt`).

---

## 06. TLS in Kubernetes

- Video reference: https://kodekloud.com/topic/tls-in-kubernetes/
- Topic: applying the general TLS/PKI concepts from file 05 specifically to a Kubernetes cluster's internal architecture.

### Server certificates vs. Client certificates

- **The two primary requirements are:**
  - All the various **services** within the cluster must use **server certificates**.
  - All **clients** must use **client certificates** to verify they are who they say they are.
- In short: **Server Certificates for Servers**, **Client Certificates for Clients.**
- **Inferred context (image `tls.PNG`):** the same image referenced in file 02 — a cluster-topology diagram with every inter-component communication link (etcd, kube-apiserver, scheduler, controller-manager, kubelet, kube-proxy, and end users) drawn as encrypted TLS connections; here in file 06 it's most likely annotated more specifically to distinguish which side of each connection is acting as the **server** (needing a server certificate to prove its identity to callers) and which side is the **client** (needing a client certificate to prove its identity to the server it's calling).

### Identifying the servers and clients in a Kubernetes cluster

- **Let's look at the different components within the k8s cluster and identify the various servers and clients and who talks to whom.**
- **Inferred context (image `certs.PNG`) — the full Kubernetes PKI/certificate tree (a key exam diagram):** based on the detailed certificate-generation walkthrough that follows in file 07, this diagram almost certainly lays out the **entire certificate map of a kubeadm cluster**, structured as:
  - **etcd** — acts purely as a **server** (and peer) from the perspective of this diagram: it needs a **server certificate** (`etcd-server.crt`/`server.crt`) to authenticate itself to clients, plus **peer certificates** for etcd-to-etcd communication in a multi-node etcd cluster.
  - **kube-apiserver** — has a **dual role**:
    - As a **server**: it needs a **server certificate** (`apiserver.crt`) presented to every client that calls it (`kubectl`, other components, end users).
    - As a **client**: it needs its own **client certificates** to authenticate itself when it calls out to *other* servers — specifically `apiserver-etcd-client.crt/key` (to talk to etcd as a client) and `apiserver-kubelet-client.crt/key` (to talk to each kubelet as a client).
  - **kube-scheduler** — a pure **client** of the kube-apiserver; needs a client certificate with `CN=system:kube-scheduler`.
  - **kube-controller-manager** — a pure **client** of the kube-apiserver; needs a client certificate with `CN=system:kube-controller-manager`. It additionally needs the **CA's root certificate and private key** itself, because it is the component responsible for signing new certificates when the Certificates API is used (covered in file 11).
  - **kube-proxy** — a pure **client** of the kube-apiserver, running on every node; needs a client certificate with `CN=system:kube-proxy`.
  - **kubelet** — has a **dual role** on every worker/control-plane node:
    - As a **server**: exposes its own HTTPS API (used by the kube-apiserver to fetch logs/exec into containers/etc.), so it needs a **server certificate**, typically named per-node (e.g. `kubelet.crt`).
    - As a **client**: it calls back into the kube-apiserver to register the node and report status, so it also needs a **client certificate**, conventionally named `system:node:<node-name>` in the group `system:nodes`.
  - **Administrators (kubectl users)** — pure **clients**; the cluster admin gets a client certificate (conventionally `CN=kube-admin`, often in group `O=system:masters` to get full cluster-admin authorization) used inside their local kubeconfig file.
  - Sitting above every one of the above is the cluster's own self-signed **root CA** (`ca.crt` / `ca.key`) — every server certificate and every client certificate above is signed by this one CA, which is exactly how every component ends up able to trust every other component's presented certificate (mirroring the general PKI trust-chain concept from file 05, just with the cluster's own CA standing in for a public CA like DigiCert).

---

## 07. TLS in Kubernetes — Certificate Creation

- Video reference: https://kodekloud.com/topic/tls-in-kubernetes-certificate-creation/
- Topic: the concrete `openssl` commands used to actually build out the certificate tree described in file 06.

### Generate Certificates

- There are different tools available for generating certificates, such as **`easyrsa`**, **`openssl`**, **`cfssl`**, or many others.

### Certificate Authority (CA)

- **Generate Keys:**
  ```
  $ openssl genrsa -out ca.key 2048
  ```
- **Generate CSR:**
  ```
  $ openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
  ```
- **Sign certificate (self-sign, since this is the root CA):**
  ```
  $ openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
  ```
- **Inferred context (image `ca1.PNG`):** likely a 3-box flow diagram matching exactly these 3 commands — `ca.key` (private key) → `ca.csr` (signing request naming `CN=KUBERNETES-CA`) → `ca.crt` (final self-signed root certificate) — establishing the cluster's own root of trust, from which every other certificate in the cluster will be signed.

### Generating Client Certificates

#### Admin User Certificate

- **Generate Keys:**
  ```
  $ openssl genrsa -out admin.key 2048
  ```
- **Generate CSR:**
  ```
  $ openssl req -new -key admin.key -subj "/CN=kube-admin" -out admin.csr
  ```
- **Sign certificate (signed by the cluster's CA this time, not self-signed):**
  ```
  $ openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
  ```
  - **Inferred context (image `ca2.PNG`):** likely the same 3-step flow style as `ca1.PNG`, but now showing the CSR being signed by the **CA's** cert+key pair (`-CA ca.crt -CAkey ca.key`) rather than self-signed — visually reinforcing the difference between the root CA (self-signed) and every other certificate in the cluster (CA-signed).
- **Certificate with admin privileges** — adding an `O=` (Organization) field to the CSR's subject to place the admin user into the built-in **`system:masters`** group, which is bound to the `cluster-admin` ClusterRole by a default ClusterRoleBinding:
  ```
  $ openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
  ```
  - **Exam-relevant:** the `CN` field becomes the **username** Kubernetes sees on the request; the `O` field becomes the **group** — this `CN`/`O` mapping is fundamental to how Kubernetes authenticates *and* authorizes based on client certificates, and is reused for every other component below.

#### Same procedure for every other client component

- **We follow the same procedure to generate client certificates for all other components that access the kube-apiserver.**
- **Inferred context (images `crt1.PNG`, `crt2.PNG`, `crt3.PNG`, `crt4.PNG`):** based on the components enumerated in file 06's certificate tree, these four images most likely walk through the identical `genrsa` → `req -new` → `x509 -req -CA ca.crt -CAkey ca.key` three-step pattern shown above for the admin user, just substituting each component's own conventional `CN`:
  - `crt1.PNG` — **kube-scheduler**: `CN=system:kube-scheduler`.
  - `crt2.PNG` — **kube-controller-manager**: `CN=system:kube-controller-manager`.
  - `crt3.PNG` — **kube-proxy**: `CN=system:kube-proxy`.
  - `crt4.PNG` — likely a summary table collecting all of the client-certificate `CN` values generated so far (admin, scheduler, controller-manager, proxy) side by side, showing the consistent `system:<component-name>` naming convention Kubernetes expects for control-plane clients so that built-in RBAC ClusterRoles (which are pre-bound to these exact usernames) apply correctly out of the box.

### Generating Server Certificates

#### ETCD Server certificate

- **Inferred context (image `etc1.PNG`):** likely shows the same `genrsa` → `req -new -subj "/CN=etcd-server"` → `x509 -req -CA ca.crt -CAkey ca.key` pattern applied to etcd's own server certificate (commonly `etcd-server.crt`/`etcd-server.key`), used by etcd to prove its identity to clients connecting to its client API port.
- **Inferred context (image `etc2.PNG`):** likely calls out the **peer certificate** requirement for etcd specifically — because etcd is frequently deployed as a multi-node cluster for high availability, each etcd instance also needs a **peer certificate** to securely authenticate the other etcd nodes it replicates data with, in addition to its server certificate for external clients; this image most likely also notes that etcd's server certificate needs **Subject Alternative Names (SANs)** covering every hostname/IP a client might use to reach that etcd node (e.g. `localhost`, `127.0.0.1`, and the node's real IP), since a plain `CN` alone is not validated by modern TLS clients for hostname matching.

#### Kube-apiserver certificate

- **Inferred context (image `api1.PNG`):** likely shows the kube-apiserver's own **server certificate** generation (commonly named `apiserver.crt`/`apiserver.key`), highlighting that because so many different names/IPs can be used to reach the API server (`kubernetes`, `kubernetes.default`, `kubernetes.default.svc`, `kubernetes.default.svc.cluster.local`, the master node's actual IP, `127.0.0.1`, and the cluster's internal Service IP e.g. `10.96.0.1`), the CSR must include **all of these as Subject Alternative Names (SANs)** — otherwise clients connecting via any name/IP not listed will get a certificate validation error.
- **Inferred context (image `api2.PNG`):** likely shows the kube-apiserver's *client-side* certificates — since the API server itself acts as a **client** when it calls out to etcd and to each kubelet, it additionally needs its own dedicated client certs: `apiserver-etcd-client.crt/key` (CN typically `kube-apiserver-etcd-client`, used when the apiserver talks to etcd) and `apiserver-kubelet-client.crt/key` (CN typically `kube-apiserver-kubelet-client`, used when the apiserver calls a kubelet's HTTPS API for logs/exec/port-forward) — matching the "dual role" described for kube-apiserver in file 06's certificate tree.

### Kubectl Nodes (Server Cert)

- **Inferred context (image `kctl1.PNG`):** likely covers the **kubelet's server certificate** — each kubelet runs its own HTTPS server (the "Kubelet API") that the kube-apiserver calls into (e.g. for `kubectl logs`/`kubectl exec`), so each node's kubelet needs a server certificate (commonly named per node, e.g. `kubelet.crt`), generated with the same `genrsa`/`req -new`/`x509 -req` pattern, ideally with SANs covering the node's hostname and IP.

### Kubectl Nodes (Client Cert)

- **Inferred context (image `kctl2.PNG`):** likely covers the **kubelet's client certificate** — used by the kubelet itself when it authenticates *to* the kube-apiserver (e.g. to register the node, report node status/heartbeats, and watch for Pods assigned to it). Convention: `CN=system:node:<node-name>`, `O=system:nodes` — the `system:nodes` group is bound by a default Kubernetes RBAC ClusterRoleBinding to the **Node Authorizer**, which is what actually restricts each kubelet to only reading/writing objects relevant to its own node (Pods scheduled to it, its own Node object, related Endpoints/Services, etc.).
- **Exam-relevant summary for the whole file:** every certificate above follows the identical 3-step `openssl` recipe — `genrsa` (private key) → `req -new -subj "/CN=..."` (CSR) → `x509 -req -CA ca.crt -CAkey ca.key` (CA-signed certificate) — with the **only** things that vary being the `CN`/`O` subject values (which determine the resulting Kubernetes username/group) and, for server certificates, the addition of SANs for every hostname/IP a client might connect through.

---

## 08. View Certificate Details

- Video reference: https://kodekloud.com/topic/view-certificate-details/
- Topic: how to locate and inspect the certificates that a running kubeadm cluster is actually using, plus how to check the logs of the components that use them (for troubleshooting cert issues).

### View Certs

- **Inferred context (image `hrd.PNG`):** likely shows the default certificate directory structure of a kubeadm-provisioned cluster, i.e. **`/etc/kubernetes/pki/`** containing `ca.crt`/`ca.key`, `apiserver.crt`/`apiserver.key`, `apiserver-etcd-client.crt/key`, `apiserver-kubelet-client.crt/key`, plus an `etcd/` subdirectory holding etcd's own `ca.crt`, `server.crt/key`, `peer.crt/key`, etc.
- **Inferred context (image `hrd1.PNG`):** likely shows how to actually *find* which certificate file a given static Pod is configured to use — by inspecting its manifest under `/etc/kubernetes/manifests/` (e.g. `kube-apiserver.yaml`, `etcd.yaml`) and reading the `--tls-cert-file`, `--tls-private-key-file`, `--client-ca-file`, `--etcd-certfile`, etc. command-line flags — exactly the technique the file 10 practice test walks through step by step.
- **To view the details of a certificate:**
  ```
  $ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
  ```
  - `-text` prints the certificate's decoded contents (Subject, Issuer, Validity dates, Subject Alternative Names, public key, signature algorithm, etc.); `-noout` suppresses printing the raw base64/PEM block, leaving just the human-readable decoded output.
  - **Inferred context (image `hrd2.PNG`):** likely shows sample decoded output of exactly this command, including fields such as `Issuer: CN = kubernetes` (or `KUBERNETES-CA`), `Subject: CN = kube-apiserver`, a `Validity` block with `Not Before`/`Not After` dates, and an `X509v3 Subject Alternative Name` extension listing DNS names like `kubernetes`, `kubernetes.default`, `kubernetes.default.svc`, `kubernetes.default.svc.cluster.local` plus IP addresses like the master node IP and the cluster Service IP — this is exactly the output the practice test in file 10 asks students to read for Issuer, Alternative Names, and Expiry.

### Follow the same procedure to identify information about all the other certificates

- **Inferred context (image `hrd3.PNG`):** likely a checklist/summary reminding you to repeat the same `openssl x509 -in <file> -text -noout` command against every other certificate file in `/etc/kubernetes/pki/` (and `/etc/kubernetes/pki/etcd/`) to audit the whole cluster's PKI — e.g. `ca.crt`, `apiserver-etcd-client.crt`, `apiserver-kubelet-client.crt`, `etcd/server.crt`, `etcd/ca.crt`, etc.

### Inspect Server Logs — Hardware setup

- If the cluster is installed **the "hard way"** (services run directly via systemd, not as static Pods), inspect server logs using **`journalctl`**:
  ```
  $ journalctl -u etcd.service -l
  ```
  - `-u etcd.service` filters logs to just that systemd unit; `-l` (`--no-pager`/full-output equivalent, shows full lines without truncation).
  - **Inferred context (image `hrd4.PNG`):** likely shows sample `journalctl` output for the `etcd.service` unit, including a plausible **TLS-related error message** (e.g. "certificate signed by unknown authority" or "x509: certificate is valid for X, not Y") — this is the classic troubleshooting scenario this lesson is building toward: a cert misconfiguration surfacing as a specific error string in the component's logs.

### Inspect Server Logs — kubeadm setup

- If the cluster was installed via **`kubeadm`** (components run as static Pods), view logs using `kubectl`:
  ```
  $ kubectl logs etcd-master
  ```
  - **Inferred context (image `hrd5.PNG`):** likely shows sample output of `kubectl logs etcd-master`, again most plausibly containing a TLS/certificate-related error line as the running example.
- If `kubectl`/the API server itself is down (so `kubectl logs` isn't usable — a classic chicken-and-egg problem when the very component you need to debug *is* the API server), fall back to inspecting the container runtime directly:
  ```
  $ docker ps -a
  $ docker logs <container-id>
  ```
  - `docker ps -a` lists all containers (including stopped/crashed ones) so you can find the crashed static-Pod container's ID; `docker logs <container-id>` then prints that specific container's stdout/stderr — the only way to see a control-plane component's logs when it has crashed so badly that even the kubelet can't be queried through the (dead) API server.
  - **Inferred context (image `hrd6.PNG`):** likely shows sample `docker ps -a` output listing an exited/crashed `etcd` or `kube-apiserver` container alongside its container ID, and then sample `docker logs <id>` output showing the underlying error (again, most plausibly a certificate-path or expiry error) — completing the "when kubectl doesn't work, drop to the container runtime" troubleshooting escalation path this lesson is teaching.

### K8s Reference Docs

- https://kubernetes.io/docs/setup/best-practices/certificates/#certificate-paths

---

*Continued in [kubernetes-security-notes_part_02.md](kubernetes-security-notes_part_02.md)
— Sections 09–15 (Certificate Health-Check Spreadsheet, Practice Test — View Certificate
Details, Certificate API, Practice Test — Certificates API, KubeConfig, Practice Test —
KubeConfig, API Groups).*
