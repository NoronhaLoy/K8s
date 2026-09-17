# Security — Complete Notes (Part 4 of 4)

> **Part 4 of 4** — covers Sections 23–30 (Image Security + Practice Test, Security
> Context + Practice Test, Network Policies + Practice Test, kubectx and kubens
> commands, Download Presentation Deck) plus the Quick Revision Checklist for the
> entire Security section.
> Previous: [kubernetes-security-notes_part_03.md](kubernetes-security-notes_part_03.md)
> (Sections 16–22).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/07-Security/`

---

## 23. Image Security

- Video reference: *Image Security* — https://kodekloud.com/topic/image-security/

### Image

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-pod
  spec:
    containers:
    - name: nginx
      image: nginx
  ```
  - **Inferred context (image `img1.PNG`):** likely explains the full image-name syntax that `image: nginx` is shorthand for — `<registry>/<user-or-account>/<image>:<tag>`, e.g. Docker Hub's default expansion is `docker.io/library/nginx`, where `docker.io` is the registry, `library` is the default "official images" account/namespace, `nginx` is the image name, and (implicitly) `latest` is the default tag if none is specified.
  - **Inferred context (image `img2.PNG`):** likely contrasts this with a fully-qualified image reference against a **different** registry, e.g. `gcr.io/kubernetes-e2e-test-images/dnsutils` — showing how the same three-part naming convention (registry / account-or-project / image) generalizes to registries other than Docker Hub, such as Google Container Registry (`gcr.io`) or a private/internal registry.

### Private Registry

- To log in to a private registry:
  ```
  $ docker login private-registry.io
  ```
- Run the application using the image available at the private registry:
  ```
  $ docker run private-registry.io/apps/internal-app
  ```
  - **Inferred context (image `prvr.PNG`):** likely a diagram showing a Docker client authenticating to a private registry (`private-registry.io`) with credentials before it's allowed to `pull`/`run` an image from it — versus a public registry which requires no login — establishing why credentials are needed at all before the Kubernetes-side `imagePullSecrets` mechanism is introduced.
- To pass the credentials to Docker (untagged) on the worker node, first create a **Secret** object with the credentials in it:
  ```
  $ kubectl create secret docker-registry regcred \
    --docker-server=private-registry.io \
    --docker-username=registry-user \
    --docker-password=registry-password \
    --docker-email=registry-user@org.com
  ```
- Then specify the secret inside the Pod definition file under the **`imagePullSecrets`** section:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-pod
  spec:
    containers:
    - name: nginx
      image: private-registry.io/apps/internal-app
    imagePullSecrets:
    - name: regcred
  ```
  - **Inferred context (image `prvr1.PNG`):** likely a flow diagram tying the two preceding pieces together — `kubectl create secret docker-registry` on one side creating a `regcred` Secret object (holding a base64-encoded `.dockercfg`/`.dockerconfigjson`), and the kubelet on a worker node reading that Secret (referenced via the Pod's `imagePullSecrets`) to authenticate its own internal image-pull request against `private-registry.io` before it can start the container — i.e. the kubelet, not the end user, performs the authenticated pull on the Pod's behalf using the Secret's embedded credentials.

#### K8s Reference Docs

- https://kubernetes.io/docs/concepts/containers/images/

---

## 24. Practice Test — Image Security

Solutions to the practice test — Image Security. Step-by-step lab-guide walkthrough:

1. **"We have an application running on our cluster. Let us explore it first. What image is the application using?"**
   ```
   $ kubectl get deploy -o wide
   ```
   - The `-o wide` output includes an `IMAGES` column showing the currently-configured container image(s) for the Deployment.

2. **"Use the `kubectl edit deployment` command to edit the image name to `myprivateregistry.com:5000/nginx:alpine`."**
   ```
   $ kubectl edit deployment web
   ```
   - Opens the Deployment's manifest in the default editor; change the `spec.template.spec.containers[].image` field to `myprivateregistry.com:5000/nginx:alpine` and save.

3. **"Run the command `kubectl get pods` and check the status of the pods."**
   ```
   $ kubectl get pods
   ```
   - At this point, since no registry credentials have been configured yet, the new Pods created by the rollout will most likely show a status like `ImagePullBackOff` or `ErrImagePull` — because `myprivateregistry.com:5000` requires authentication and none has been supplied.

4. **"Run command `kubectl create secret docker-registry private-reg-cred --docker-username=dock_user --docker-password=dock_password --docker-server=myprivateregistry.com:5000 --docker-email=dock_user@myprivateregistry.com`."**
   ```
   $ kubectl create secret docker-registry private-reg-cred --docker-username=dock_user --docker-password=dock_password --docker-server=myprivateregistry.com:5000 --docker-email=dock_user@myprivateregistry.com
   ```
   - Creates a `docker-registry`-type Secret named `private-reg-cred` holding the credentials needed to authenticate against `myprivateregistry.com:5000`.

5. **"Edit deployment using `kubectl edit deploy web` command and add `imagePullSecrets` section. Use `private-reg-cred`."**
   ```
   $ kubectl edit deploy web
   ```
   - Add, under `spec.template.spec`:
     ```yaml
     imagePullSecrets:
     - name: private-reg-cred
     ```

6. **"Check the status of PODs. Wait for them to be running. You have now successfully configured a Deployment to pull images from the private registry."**
   ```
   $ kubectl get pods
   ```
   - The Pods should now transition to `Running`, confirming that the kubelet successfully authenticated the image pull against the private registry using the credentials referenced via `imagePullSecrets`.

**Exam-relevant takeaway:** an `ImagePullBackOff`/`ErrImagePull` status right after switching to a private-registry image is the classic signal that `imagePullSecrets` is missing (or the credentials in the referenced Secret are wrong). The fix is always the same two-step pattern: `kubectl create secret docker-registry <name> --docker-server=... --docker-username=... --docker-password=... --docker-email=...` followed by adding `imagePullSecrets: [{name: <name>}]` under the Pod template spec.

---

## 25. Security Context

- Video reference: *Security Contexts* — https://kodekloud.com/topic/security-contexts-2/

### Container Security

  ```
  $ docker run --user=1001 ubuntu sleep 3600
  $ docker run -cap-add MAC_ADMIN ubuntu
  ```
  - `--user=<uid>` runs the container's process as a specific (non-root) user ID.
  - `--cap-add <CAPABILITY>` grants an additional **Linux capability** (a fine-grained slice of root's privileges — e.g. `MAC_ADMIN` controls Mandatory Access Control settings) to a container that would otherwise run with a restricted default capability set, without granting it full root/privileged access.
  - **Inferred context (image `csec.PNG`):** likely a comparison diagram of Docker's default limited capability set (a curated subset of the full ~40 Linux capabilities that `root` normally has, e.g. `CHOWN`, `NET_BIND_SERVICE`, etc.) versus the full list of possible capabilities, with `--cap-add`/`--cap-drop` shown as the mechanism to add back (or remove) specific ones — plus `--privileged`, which grants **all** capabilities at once, contrasted as the "everything" extreme.

### Kubernetes Security

- You may choose to configure the security settings at a **container level** or at a **pod level**.
  - **Inferred context (image `ksec.PNG`):** likely a Pod-diagram showing a `securityContext` field that can be attached either at `spec.securityContext` (applies to the whole Pod / all its containers, as a default) or at `spec.containers[].securityContext` (applies to just that one container, and **overrides** any pod-level setting for that specific field) — directly matching the two YAML variants demonstrated next in this same file.

### Security Context

- To add a security context on the container **or pod**, use a field called **`securityContext`** under the `spec` section.
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: web-pod
  spec:
    securityContext:
      runAsUser: 1000
    containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
  ```
  - **Inferred context (image `sxc1.PNG`):** likely illustrates this Pod-level `securityContext` applying its `runAsUser: 1000` setting as the **default** for every container in the Pod (since there's only one container here, it simply runs as UID 1000).
- To set the same context at the **container level** instead, move the whole section under the container's own spec:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: web-pod
  spec:
    containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
  ```
  - **Inferred context (image `sxc2.PNG`):** likely shows the same effect (container runs as UID 1000) but achieved via a **container-level** `securityContext`, and — as reinforced in file 26's practice-test notes — a container-level `runAsUser` **overrides** any Pod-level `runAsUser` for that specific container, while a Pod-level setting still applies as the default to any *other* containers in the same Pod that don't define their own override.
- To add **capabilities**, use the **`capabilities`** option (note: capabilities can **only** be configured at the **container** level, not the Pod level — this is standard Kubernetes knowledge reflected in the placement of the field in the example below):
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: web-pod
  spec:
    containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
  ```
  - **Inferred context (image `cap.PNG`):** likely a list/diagram of the `capabilities.add`/`capabilities.drop` sub-fields, showing example capability names (`MAC_ADMIN`, `NET_ADMIN`, `SYS_TIME`, `CHOWN`, etc.) that can be individually added to or dropped from a container's Linux capability set — this exact `capabilities.add: ["SYS_TIME"]` pattern reappears in file 26's practice test to let a container change the system clock.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/configure-pod-container/security-context/

---

## 26. Practice Test — Security Context

Solutions to practice test — Security Context. Step-by-step lab-guide walkthrough:

1. **"Run the command `kubectl exec ubuntu-sleeper -- whoami` and count the number of pods."**
   ```
   $ kubectl exec ubuntu-sleeper whoami
   ```
   - Reports the effective user the container's process is running as (default is typically `root` unless a `securityContext.runAsUser` has been set).

2. **"Set a security context to run as user 1010."**
   ```
   $ kubectl get pods ubuntu-sleeper -o yaml > ubuntu.yaml
   $ kubectl delete pod ubuntu-sleeper
   $ vi ubuntu.yaml   # (add securityContext section)
       securityContext:
         runAsUser: 1010
   $ kubectl create -f ubuntu.yaml
   ```
   - Since `securityContext.runAsUser` is immutable on a running Pod, the standard technique is: **export** the existing Pod's YAML to a file, **delete** the running Pod, **edit** the exported file to add/change the `securityContext` block, then **recreate** the Pod from the edited file.

3. **"The User ID defined in the securityContext of the container overrides the User ID in the POD."**
   - Confirmed as true/correct — a **container-level** `runAsUser` takes precedence over a **Pod-level** `runAsUser` for that specific container.

4. **"The User ID defined in the securityContext of the POD is carried over to all the PODs in the container."**
   - Confirmed as true/correct (worded slightly loosely in the source, but the intent is): a **Pod-level** `runAsUser` setting is inherited as the default by **all containers** within that Pod (unless a given container overrides it with its own container-level `securityContext.runAsUser`).

5. **"Run `kubectl exec -it ubuntu-sleeper -- date -s '19 APR 2012 11:14:00'`."**
   ```
   $ kubectl exec -it ubuntu-sleeper -- date -s '19 APR 2012 11:14:00'
   ```
   - Attempts to change the system date/time from inside the container. This will fail with a permission error unless the container has the `SYS_TIME` Linux capability (changing the system clock requires elevated privilege beyond a normal unprivileged process).

6. **"Add `SYS_TIME` capability to the container's securityContext."**
   ```
   $ kubectl get pods ubuntu-sleeper -o yaml > ubuntu.yaml
   $ kubectl delete pod ubuntu-sleeper
   $ vi ubuntu.yaml
   ```
   Under the container section add:
   ```yaml
   securityContext:
       capabilities:
         add: ["SYS_TIME"]
   ```
   ```
   $ kubectl create -f ubuntu.yaml
   ```
   - Same export → delete → edit → recreate pattern as step 2, but this time adding a `capabilities.add` list at the **container** level (capabilities cannot be set at Pod level).

7. **"Now try to run the below command in the pod to set the date. If the security capability was added correctly, it should work. If it doesn't, make sure you changed the user back to root."**
   ```
   $ kubectl exec -it ubuntu-sleeper -- date -s '19 APR 2012 11:14:00'
   ```
   - If this now succeeds, it confirms the `SYS_TIME` capability was added correctly. The caveat about "changing the user back to root" is a reminder that even with `SYS_TIME` granted, running as a **non-root** `runAsUser` (e.g. the `1010` set earlier) can still block privileged operations like setting the system clock — capabilities alone don't always substitute for the right UID, depending on how the underlying binary/kernel checks permissions, so both settings need to be reconciled for the command to actually work.

**Exam-relevant takeaway:** neither `runAsUser` nor `capabilities` changes can be applied to a running Pod in place — the standard exam pattern is always "get -o yaml > file, delete the pod, edit the file, recreate from the file." Container-level `securityContext` fields override Pod-level ones for that same container; Pod-level fields act as the default for containers that don't specify their own. `capabilities` (add/drop) can only be set at the container level, never at the Pod level.

---

## 27. Network Policies

- Video reference: *Network Policies* — https://kodekloud.com/topic/network-policies-3/

### Traffic flowing through a webserver serving frontend to users, an app server serving backend API, and a database server

- **Inferred context (image `traffic.PNG`):** likely a 3-tier architecture diagram — `[User] → [Web/Frontend Pod] → [App/Backend API Pod] → [Database Pod]` — showing traffic flowing left-to-right through each tier, used to set up the Ingress/Egress vocabulary that follows: from the App server's perspective, traffic arriving from the Web server is **Ingress**, and traffic it sends onward to the Database is **Egress**.
- There are two types of traffic:
  - **Ingress**
  - **Egress**
  - **Inferred context (image `ing1.PNG`):** likely defines **Ingress** as traffic **coming into** a Pod/resource from elsewhere (e.g. the Database Pod receiving a connection from the App server is Ingress traffic, *from the Database's point of view*).
  - **Inferred context (image `ing2.PNG`):** likely defines **Egress** as traffic **leaving** a Pod/resource, going out to another destination (e.g. the App server initiating a connection out to the Database is Egress traffic, *from the App server's point of view*) — together `ing1`/`ing2` establish that Ingress/Egress are always relative to the resource being described, not absolute directions.

### Network Security

- **Inferred context (image `nsec.PNG`):** likely shows that, **by default, Kubernetes networking is "all-allow"** — any Pod can reach any other Pod (and any Pod can reach any external endpoint) with no restrictions at all, unless something explicitly restricts it — setting up the motivation for NetworkPolicy objects, e.g. restricting the Database tier so that **only** the App server (not the Web server, and not arbitrary other Pods) can connect to it on its DB port.

### Network Policy

- **Inferred context (image `npol.PNG`):** likely introduces the **NetworkPolicy** object itself as the mechanism to enforce such restrictions — a Kubernetes API object of `kind: NetworkPolicy` that gets **linked to one or more Pods via a label selector**, similar in spirit to how a Service is linked to Pods.
- **Inferred context (image `npol1.PNG`):** likely reiterates the important caveat that a NetworkPolicy is only **enforced** if the underlying **network plugin/CNI solution supports NetworkPolicies** — e.g. Kubernetes' default "flat" networking model and simple CNI plugins (like an unconfigured basic bridge plugin) do **not** implement NetworkPolicy enforcement at all; solutions like **Calico**, **Cilium**, **Weave Net**, and others do. Without a NetworkPolicy-supporting CNI, NetworkPolicy objects can be created and will sit in the API silently, but they will have **no actual effect** on traffic.

### Network Policy Selectors

- **Inferred context (image `npolsec.PNG`):** likely shows how a NetworkPolicy selects **which Pods it applies to** via `spec.podSelector` (a label selector, matching the target/protected Pods — e.g. the Database Pods labeled `role: db`), and separately, within its ingress/egress rules, selects **which sources/destinations are allowed** via nested selectors such as `podSelector` (other Pods by label), `namespaceSelector` (entire namespaces by label), or `ipBlock` (CIDR ranges) — the same three selector types combinable within a single rule's `from`/`to` list.

### Network Policy Rules

- **Inferred context (image `npol2.PNG`):** likely shows the anatomy of an ingress/egress rule block — each rule has a `from` (for ingress) or `to` (for egress) list of allowed peers, plus a `ports` list restricting which protocol/port combinations are allowed — and clarifies that multiple entries **within the same `from`/`to` list item** are logically ANDed together (e.g. a Pod matching a `podSelector` **and** a `namespaceSelector` in the same list element), whereas multiple **separate** list items are logically ORed (any one of them matching is sufficient).

### Create network policy

- To create a network policy:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
   name: db-policy
  spec:
    podSelector:
      matchLabels:
        role: db
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            role: api-pod
      ports:
      - protocol: TCP
        port: 3306
  ```
  ```
  $ kubectl create -f policy-definition.yaml
  ```
  - This policy targets Pods labeled `role: db` (e.g. the database tier) and allows **Ingress** traffic **only** from Pods labeled `role: api-pod`, and only on **TCP port 3306** (the MySQL port) — matching exactly the 3-tier scenario introduced at the top of this file: the App server (`api-pod`) is allowed to reach the Database (`db`), but nothing else is (e.g. the Web server is implicitly blocked from directly reaching the Database, since it isn't `role: api-pod`).
  - **Inferred context (image `npol3.PNG`):** likely a visual rendering of exactly this YAML applied to the 3-tier diagram from `traffic.PNG` — showing the `db-policy` "wrapping" the Database Pod, with a green arrow allowed in from the `api-pod`-labeled App server and a red/blocked arrow from the Web server (or any other Pod) attempting to reach the Database directly.
  - **Inferred context (image `npol4.PNG`):** likely reiterates/summarizes the same policy's effect in tabular or bullet form (target Pods / allowed direction / allowed peers / allowed ports), as a recap slide.

### Note

- **Inferred context (image `note1.PNG`):** likely calls out one or more important gotchas commonly paired with this exact lesson in the CKA course, most plausibly: (1) once a NetworkPolicy selects a Pod for a given `policyTypes` direction (e.g. `Ingress`), **all traffic in that direction not explicitly allowed by some rule is denied** — i.e. NetworkPolicies are default-deny-once-selected, allow-list based; and (2) if **no** NetworkPolicy selects a given Pod at all, that Pod remains fully open (all traffic allowed) in whichever direction(s) aren't covered by any policy.

#### Additional lecture

- Additional lecture on [Developing Networking Policies](https://kodekloud.com/topic/developing-network-policies/)

#### K8s Reference Docs

- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/

---

## 28. Practice Test — Network Policies

Solutions to practice test — Network Policies. Step-by-step lab-guide walkthrough:

1. **"Run the command `kubectl get networkpolicy`."**
   ```
   $ kubectl get networkpolicy
   ```
   - Lists existing NetworkPolicy objects in the current namespace — establishes the baseline of what's already configured.

2. **"Run the command `kubectl get networkpolicy`."** *(repeated inspection)*
   ```
   $ kubectl get networkpolicy
   ```

3. **"Run the command `kubectl get networkpolicy` and look under pod selector."**
   ```
   $ kubectl get networkpolicy
   ```
   - Use `kubectl describe networkpolicy <name>` (or `-o yaml`) to actually read the `podSelector` field, identifying which Pods (by label) the policy targets/protects.

4. **"Run the command `kubectl describe networkpolicy` and look under PolicyTypes."**
   ```
   $ kubectl describe networkpolicy
   ```
   - The `Policy Types` field (e.g. `Ingress`, or `Ingress, Egress`) tells you which traffic direction(s) this policy actually governs — a policy listing only `Ingress` has **no effect whatsoever** on that Pod's outbound (egress) traffic.

5. **"What is the impact of the rule configured on this Network Policy?"**
   - **Answer:** *Traffic from internal to payroll pod is blocked.*
   - Reasoning: the described policy's `ingress` rules do not include the `internal` Pod(s)/namespace as an allowed source for the `payroll` Pod, so by the default-deny-once-selected behavior (file 27's "Note" section), that traffic is blocked.

6. **"What is the impact of the rule configured on this Network Policy?"**
   - **Answer:** *Internal pod can access port 8080 on payroll pod.*
   - Reasoning: a different rule (or a different port entry within the ingress rules) does explicitly allow the `internal` Pod to reach the `payroll` Pod, but only on port `8080` specifically — other ports remain blocked.

7. **"Access the UI of these applications using the link given above the terminal."**
   - Manual/interactive step — no command; use the lab's UI link to visually confirm application reachability from a browser perspective.

8. **"Only internal applications can access payroll service."**
   - A statement/assertion confirming the intended design of the policy under test (i.e. the practice test is validating that the student's or the pre-existing policy's configuration correctly restricts payroll access to only the `internal` application tier).

9. **"Perform a connectivity test using the User Interface of the Internal Application to access the `external-service` at port `8080`."**
   - **Answer:** `Successful`
   - Since no NetworkPolicy in this scenario restricts the `internal` Pod's **egress**, or none restricts `external-service`'s **ingress** from `internal`, the connection succeeds — demonstrating that NetworkPolicies only restrict what they explicitly select/cover, and traffic not addressed by any policy remains allowed by default.

10. **"Answer file located at `/var/answers/answer-internal-policy.yaml`."**
    ```
    $ kubectl create -f /var/answers/answer-internal-policy.yaml
    ```
    - Applies the pre-written answer manifest — presumably a NetworkPolicy that correctly restricts traffic per the lab's requirements (e.g. locking down `payroll` so only `internal`, on the correct port, can reach it, while leaving `external-service` access unaffected).

**Exam-relevant takeaway:** always check both `podSelector` (who's protected) **and** `policyTypes` (which direction(s) are actually governed) — a policy that only lists `Ingress` leaves Egress completely open regardless of what ingress rules say, and vice versa. Connectivity that isn't explicitly covered by any matching NetworkPolicy rule remains allowed by Kubernetes' default "no restrictions" networking model; restrictions only kick in for traffic direction(s)/peers a policy actually addresses.

---

## 29. kubectx and kubens commands (Optional)

- This is an **optional** lesson.
- Video reference: *kubectx and kubens command line utilities* — https://kodekloud.com/topic/kubectx-and-kubens-command-line-utilities/
- **This source file contains no inline written content beyond the title and the video link** — it is a placeholder page pointing to a hands-on/optional tutorial video, with no accompanying text, YAML, or commands captured in the docs export.
- **Inferred context (video: "kubectx and kubens command line utilities") — likely content, based on the well-known purpose of these two community tools:**
  - `kubectx` is a third-party CLI utility for quickly **switching between kubeconfig contexts** (clusters/users/namespaces bundles), avoiding the more verbose `kubectl config use-context <name>` / `kubectl config get-contexts` commands.
  - `kubens` is a companion utility for quickly **switching the active/default namespace** within the current context, avoiding the more verbose `kubectl config set-context --current --namespace=<ns>`.
  - Both are optional convenience tools (not part of core `kubectl`), typically installed separately (e.g. via a package manager or as a `kubectl` plugin via `krew`), and are commonly demoed together since they solve closely related "which cluster/namespace am I currently operating against" friction points.
- **Because there is no original written text to quote beyond the title/link, no verbatim commands from this specific file can be preserved.**

---

## 30. Download Presentation Deck

- The section provides a downloadable slide deck accompanying the video lectures: [Presentation Deck](https://kodekloud.com/topic/download-presentation-deck-6/).
- Purely a resource/reference link — no technical content of its own (identical in nature/format to the equivalent "Download Presentation Deck" placeholder pages seen at the end of other sections in this course, e.g. the Cluster Maintenance section's file 11).

---

## Quick Revision Checklist

- [ ] **Authentication mechanisms**
  - Static Password/Token files (`--basic-auth-file`) — deprecated/removed (v1.19+), plaintext, educational only.
  - Client certificates — `CN` becomes the **username**, `O` becomes the **group** (e.g. `CN=kube-admin/O=system:masters`).
  - External identity services (LDAP/Kerberos/OIDC/SSO) for production-grade setups.
  - Every request — from `kubectl`, another component, or a raw `curl` — is authenticated (and then authorized) by the **kube-apiserver**; there is no other entry point.

- [ ] **TLS / PKI in Kubernetes**
  - Server certs for servers, client certs for clients: **etcd** (server+peer), **kube-apiserver** (dual: server to callers, client to etcd/kubelets), **scheduler/controller-manager/kube-proxy** (pure clients, `CN=system:<component>`), **kubelet** (dual: server for its own API, client `CN=system:node:<name>`/`O=system:nodes` to register with the apiserver).
  - Cert generation recipe (`openssl`): `genrsa` (key) → `req -new -subj "/CN=..."` (CSR) → `x509 -req -CA ca.crt -CAkey ca.key` (CA-signed cert); the root CA itself is self-signed (`-signkey` instead of `-CA`/`-CAkey`).
  - Server certs need **SANs** covering every hostname/IP clients may use to connect.
  - **etcd has its own separate CA** (`/etc/kubernetes/pki/etcd/ca.crt`), distinct from the main cluster CA (`/etc/kubernetes/pki/ca.crt`) — a classic exam trap when configuring `--etcd-cacert`/`--etcd-certfile`/`--etcd-keyfile` on the apiserver.
  - Inspect any cert: `openssl x509 -in <path> -text -noout` → read `Issuer`, `Subject`, `Validity` (`Not Before`/`Not After`), and `X509v3 Subject Alternative Name`.
  - Find which file a component uses by reading its static Pod manifest under `/etc/kubernetes/manifests/` (`kube-apiserver.yaml`, `etcd.yaml`) for the relevant `--tls-cert-file`/`--cert-file`/`--etcd-certfile`/etc. flag.
  - Troubleshooting: `journalctl -u <service> -l` (hard-way installs) or `kubectl logs <static-pod>` (kubeadm) — and if the apiserver itself is down, fall back to `docker ps -a` + `docker logs <id>`.

- [ ] **Kubernetes Certificates API**
  - Workflow: generate key+CSR with `openssl` → base64-encode the CSR → embed in a `CertificateSigningRequest` object → `kubectl create -f` → `kubectl get csr` → `kubectl certificate approve <name>` (or `deny`) → extract+`base64 --decode` the signed cert from `status.certificate`.
  - The **kube-controller-manager** performs the actual signing, using `--cluster-signing-cert-file`/`--cluster-signing-key-file` pointing at the cluster CA.
  - Not every CSR in `kubectl get csr` is a new-user request — kubelet **TLS bootstrapping** generates its own; deny+delete anything unrecognized or requesting suspicious `usages`/`groups`.

- [ ] **kubeconfig**
  - 3 sections: **clusters** (server + CA), **users** (client cert/key or token), **contexts** (cluster+user+optional namespace, conventionally named `<user>@<cluster>`).
  - Default location `$HOME/.kube/config`; override with `--kubeconfig <file>`.
  - `kubectl config view [--kubeconfig <file>]` to read it; `kubectl config [--kubeconfig <file>] use-context <name>` to switch.
  - Cert fields can be a **file path** (`client-certificate:`) or **inline base64** (`client-certificate-data:`) — mixing up a wrong file path is a classic fix-it exam task (find the real file with `ls`/`pwd`, correct the kubeconfig, verify with `kubectl get pods`).

- [ ] **API Groups**
  - **Core group** (`/api/v1`) — foundational resources (Pods, Services, ConfigMaps, Secrets, Nodes, etc.), flat, no sub-group name.
  - **Named groups** (`/apis/<group>/<version>`) — where all new features land (`apps`, `networking.k8s.io`, `rbac.authorization.k8s.io`, `certificates.k8s.io`, `storage.k8s.io`, etc.).
  - Direct API access needs client cert auth (`curl --key --cert --cacert`); `kubectl proxy` gives a cert-free local HTTP proxy on `localhost:8001` using your kubeconfig's credentials.
  - **Don't confuse** `kubectl proxy` (client-side API access helper) with **`kube-proxy`** (the per-node Service-networking component) — same word, unrelated roles.

- [ ] **Authorization modes**
  - **Node** (kubelet-specific), **ABAC** (static JSON policy file, legacy), **RBAC** (Role/RoleBinding objects, modern default), **Webhook** (delegates to an external service, e.g. OPA).
  - `--authorization-mode=Node,RBAC,Webhook` — modes are checked **in order**; the request is authorized as soon as **any** mode approves it; denied only if **none** approve.

- [ ] **RBAC — Roles & RoleBindings (namespaced)**
  - `Role` = `apiGroups`/`resources`/`verbs` (+ optional `resourceNames` to restrict to specific named objects).
  - `RoleBinding` links a `subject` (User/Group/ServiceAccount) to a `roleRef` (Role) — both objects live inside **one namespace**.
  - `kubectl get/describe role[binding] [-n <ns>] [--all-namespaces]`; `kubectl auth can-i <verb> <resource> [--as <user>] [--namespace <ns>]` to test permissions (including impersonation).

- [ ] **ClusterRoles & ClusterRoleBindings (cluster-scoped)**
  - Same shape as Role/RoleBinding but apply to **cluster-scoped resources** (nodes, PVs, storageclasses, namespaces themselves, CSRs, clusterroles) — never ask/answer "which namespace" for these, they have none.
  - Can also be used to grant access to **namespaced** resource types **across all namespaces** at once (cluster-wide admin-style grants).
  - `cluster-admin` built-in ClusterRole = `*` verbs on `*` resources in `*` groups, conventionally bound to the `system:masters` group.

- [ ] **Service Accounts**
  - For **bots/applications** (not humans); every namespace gets a `default` ServiceAccount auto-assigned to Pods unless `spec.serviceAccountName` overrides it.
  - Token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/{token,ca.crt,namespace}` (via `automountServiceAccountToken`, default `true`).
  - **v1.22+**: tokens are audience-bound, time-bound (1hr default), object-bound, delivered via a **projected volume** and the **TokenRequest API**.
  - **v1.24+ (KEP-2799)**: no more auto-created long-lived Secret tokens — use `kubectl create token <serviceaccount>` to mint one on demand; manual Secret-based tokens (`type: kubernetes.io/service-account-token` + `kubernetes.io/service-account.name` annotation) are now a fallback, not the default.

- [ ] **Image Security**
  - Image name syntax: `<registry>/<account>/<image>:<tag>` — Docker Hub defaults to `docker.io/library/<image>:latest`.
  - Private registries: `kubectl create secret docker-registry <name> --docker-server=... --docker-username=... --docker-password=... --docker-email=...`, then reference it via `imagePullSecrets: [{name: <name>}]` in the Pod spec.
  - `ImagePullBackOff`/`ErrImagePull` right after switching to a private image = missing/wrong `imagePullSecrets`.

- [ ] **Security Context**
  - `securityContext` can be set at **Pod level** (`spec.securityContext`, default for all containers) or **container level** (`spec.containers[].securityContext`, overrides Pod-level for that container).
  - `runAsUser` (UID) settable at either level; `capabilities.add`/`capabilities.drop` (Linux capabilities like `MAC_ADMIN`, `SYS_TIME`) **only** settable at the **container** level.
  - Both fields are **immutable on a running Pod** — always `get -o yaml > file`, `delete pod`, edit, `create -f file`.

- [ ] **Network Policies**
  - Kubernetes networking is **all-allow by default** — NetworkPolicy is the only way to restrict Pod-to-Pod traffic.
  - `NetworkPolicy.spec.podSelector` picks which Pods are **protected**; `ingress[].from`/`egress[].to` (with nested `podSelector`/`namespaceSelector`/`ipBlock`) picks **allowed peers**; `ports` restricts protocol/port.
  - **Requires a NetworkPolicy-capable CNI** (Calico, Cilium, Weave Net, etc.) — otherwise the object is accepted by the API but has **zero enforcement effect**.
  - Once a Pod is selected for a `policyTypes` direction (`Ingress`/`Egress`), all **unlisted** traffic in that direction is denied by default; directions **not** listed in `policyTypes` remain fully open regardless of any `ingress`/`egress` rules present.
  - `kubectl get/describe networkpolicy` — always check both `podSelector` (who's protected) and `Policy Types` (which direction(s) actually apply).

- [ ] **kubectx / kubens (optional tools)**
  - Third-party convenience CLIs: `kubectx` for switching kubeconfig contexts, `kubens` for switching the active namespace — neither is part of core `kubectl`.

---

*End of Security notes.*
