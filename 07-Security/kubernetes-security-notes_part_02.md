# Security — Complete Notes (Part 2 of 4)

> **Part 2 of 4** — covers Sections 09–15 (Certificate Health-Check Spreadsheet, Practice
> Test — View Certificate Details, Certificate API, Practice Test — Certificates API,
> KubeConfig, Practice Test — KubeConfig, API Groups).
> Previous: [kubernetes-security-notes_part_01.md](kubernetes-security-notes_part_01.md)
> (Sections 01–08).
> Next: [kubernetes-security-notes_part_03.md](kubernetes-security-notes_part_03.md)
> (Sections 16–22).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/07-Security/`

---

## 09. Certificate Health-Check Spreadsheet

- Video reference/resource: *Certificate Health-Check Spreadsheet* — https://kodekloud.com/topic/certificate-health-check-spreadsheet/
- **This source file contains no inline technical content whatsoever** — it consists solely of the page title and a single link out to a downloadable spreadsheet resource.
- This is the same pattern as the "Download Presentation Deck" placeholder pages seen at the end of other sections in this course (e.g. the Cluster Maintenance section) — a pure external-resource pointer with nothing to extract or summarize.
- **Inferred purpose of the spreadsheet (not stated in the source text, but implied by the immediately surrounding lessons on locating/inspecting certificate files):** most likely a reference table/checklist an administrator can use to systematically go through every certificate in a kubeadm cluster (`ca.crt`, `apiserver.crt`, `apiserver-etcd-client.crt`, `apiserver-kubelet-client.crt`, `front-proxy-ca.crt`, `etcd/ca.crt`, `etcd/server.crt`, `etcd/peer.crt`, each kubelet's client/server cert, etc.), noting each one's expected file path, expected `CN`/SANs, and expiry date — a practical companion tool for the certificate-troubleshooting skills taught in file 08 and exercised in file 10's practice test, and generally useful for pre-empting certificate-expiry outages (a well-known real-world kubeadm pain point, since kubeadm certs default to a 1-year validity period unless explicitly renewed).

---

## 10. Practice Test — View Certificate Details

- Practice test reference: https://kodekloud.com/topic/practice-test-view-certificate-details/
- Full step-by-step lab-guide walkthrough of the solutions:

1. **"Identify the certificate file used for the kube-api server."**
   ```
   $ cat /etc/kubernetes/manifest/kube-apiserver.yaml
   ```
   - Look for the `--tls-cert-file` flag in the container's command args.
   - **Answer: `/etc/kubernetes/pki/apiserver.crt`**

2. **"Identify the Certificate file used to authenticate kube-apiserver as a client to ETCD Server."**
   ```
   $ cat /etc/kubernetes/manifest/kube-apiserver.yaml
   ```
   - Look for the `--etcd-certfile` flag (the apiserver's *client* cert when talking to etcd, distinct from its own server cert used in step 1).
   - **Answer: `/etc/kubernetes/pki/apiserver-etcd-client.crt`**

3. **"Look for the `kubelet-client-key` option in the file `/etc/kubernetes/manifests/kube-apiserver.yaml`."**
   - Look for the `--kubelet-client-key` flag.
   - **Answer: `/etc/kubernetes/pki/apiserver-kubelet-client.key`**

4. **"Look for the cert file option in the file `/etc/kubernetes/manifests/etcd.yaml`."**
   - Look for the `--cert-file` flag inside etcd's own static Pod manifest.
   - **Answer: `/etc/kubernetes/pki/etcd/server.crt`**

5. **"Look for the CA Certificate in file `/etc/kubernetes/manifests/etcd.yaml`."**
   - Look for the `--trusted-ca-file` (or `--peer-trusted-ca-file`) flag.
   - **Answer: `/etc/kubernetes/pki/etcd/ca.crt`**

6. **"Run the command `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text`."**
   ```
   $ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text
   ```
   - Establishes the baseline decoded-output view of the apiserver's certificate used to answer the next several questions.

7. **"Run the command `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text` and look for issuer."**
   ```
   $ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text
   ```
   - Read the `Issuer:` field in the decoded output — reveals which CA signed this certificate (on a kubeadm cluster, typically `CN = kubernetes` / the cluster's own CA).

8. **"Run the command `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text` and look at Alternative Names."**
   ```
   $ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text
   ```
   - Read the `X509v3 Subject Alternative Name:` extension — lists every DNS name/IP the apiserver certificate is valid for (`kubernetes`, `kubernetes.default`, `kubernetes.default.svc`, `kubernetes.default.svc.cluster.local`, master node IP, cluster Service IP, etc.).

9. **"Run the command `openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text` and look for Subject CN."**
   ```
   $ openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text
   ```
   - Read the `Subject: CN = ...` field — identifies the certificate's own claimed identity (etcd's server certificate).

10. **"Run the command `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text` and check the Expiry date."**
    ```
    $ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text
    ```
    - Read the `Validity` block's `Not After :` field — the certificate's expiry date/time. **Exam-relevant:** kubeadm-generated certificates default to roughly a **1-year** validity, so knowing how to read this field is directly tied to real-world cert-expiry troubleshooting.

11. **"Run the command `openssl x509 -in /etc/kubernetes/pki/ca.crt -text` and look for validity."**
    ```
    $ openssl x509 -in /etc/kubernetes/pki/ca.crt -text
    ```
    - Read the `Validity` block on the **root CA** certificate specifically (note: distinct file from the apiserver cert checked previously) — the CA's own certificate typically has a much longer validity window (kubeadm defaults to 10 years for the CA) than the leaf certificates it signs.

12. **"Inspect the `--cert-file` option in the manifests file."**
    ```
    $ vi /etc/kubernetes/manifests/etcd.yaml
    ```
    - Open the etcd static Pod manifest directly in an editor to inspect its full set of TLS-related flags in context (rather than grepping for a single flag as in earlier steps).

13. **"ETCD has its own CA. The right CA must be used for the ETCD-CA file in `/etc/kubernetes/manifests/kube-apiserver.yaml`."**
    - The task is to fix a misconfigured `--etcd-cacert` flag inside the kube-apiserver manifest so that it points at etcd's own CA certificate (`/etc/kubernetes/pki/etcd/ca.crt`), rather than, e.g., mistakenly pointing at the main cluster CA (`/etc/kubernetes/pki/ca.crt`) — **etcd maintains a separate CA from the main Kubernetes cluster CA**, and mixing the two up is a classic exam trap that causes the apiserver to fail TLS verification against etcd.
    ```
    View answer at /var/answers/kube-apiserver.yaml
    ```
    - The lab environment provides a reference/known-good copy of the manifest at `/var/answers/kube-apiserver.yaml` to check your fix against.

**Exam-relevant takeaway:** every one of these lookups follows the same two-step pattern — (1) find the *file path* a given certificate/flag points to by reading the relevant static Pod manifest under `/etc/kubernetes/manifests/` (`kube-apiserver.yaml` or `etcd.yaml`), then (2) decode that certificate's contents with `openssl x509 -in <path> -text [-noout]` to answer questions about Issuer, Subject CN, Subject Alternative Names, or Validity/Expiry dates. Remember that **etcd has its own separate CA** (`/etc/kubernetes/pki/etcd/ca.crt`) distinct from the main cluster CA (`/etc/kubernetes/pki/ca.crt`), and the apiserver needs *both* the correct etcd CA (`--etcd-cacert`) and its own dedicated client cert/key pair (`--etcd-certfile`/`--etcd-keyfile`, i.e. `apiserver-etcd-client.crt/key`) to talk to etcd successfully.

---

## 11. Certificate API

- Video reference: https://kodekloud.com/topic/certificates-api/
- Topic: instead of manually generating and CA-signing certificates yourself with raw `openssl` commands (as in file 07), Kubernetes exposes a **built-in Certificates API** that lets the cluster's own CA sign certificates for you, via a Kubernetes-native CSR workflow.

### CA (Certificate Authority)

- **The CA is really just the pair of key and certificate files that we have generated** (i.e. `ca.key` + `ca.crt` from file 07) — **whoever gains access to these pair of files can sign any certificate for the kubernetes environment.**
  - **Exam/security-relevant callout:** this makes the CA's private key (`ca.key`) one of the single most sensitive files in the entire cluster — anyone holding it can mint a certificate for *any* identity (including `system:masters`), fully impersonating a cluster admin.

### Kubernetes has a built-in certificates API that can do this for you

- With the certificate API, you send a **Certificate Signing Request (CSR)** directly to Kubernetes through an API call, rather than running `openssl x509 ... -CA ca.crt -CAkey ca.key` yourself.
- **Inferred context (image `csr.PNG`):** likely a flow diagram showing a user generating a CSR locally, sending it to the Kubernetes API as a `CertificateSigningRequest` object, and an administrator (or automated approver) approving it through `kubectl`, at which point the **kube-controller-manager** — which holds the CA's key/cert internally — actually performs the signing on the cluster's behalf.

### This certificate can then be extracted and shared with the user

- **A user first creates a key:**
  ```
  $ openssl genrsa -out jane.key 2048
  ```
- **Generates a CSR:**
  ```
  $ openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
  ```
- **Sends the request to the administrator**, and the administrator takes the CSR and creates a Kubernetes `CertificateSigningRequest` object, embedding the base64-encoded CSR contents:
  ```yaml
  apiVersion: certificates.k8s.io/v1beta1
  kind: CertificateSigningRequest
  metadata:
    name: jane
  spec:
    groups:
    - system:authenticated
    usages:
    - digital signature
    - key encipherment
    - server auth
    request:
      <certificate-goes-here>
  ```
  - The `<certificate-goes-here>` placeholder is where the base64-encoded contents of `jane.csr` go, obtained via:
    ```
    $ cat jane.csr | base64
    ```
  - The object is then created in the cluster:
    ```
    $ kubectl create -f jane.yaml
    ```
  - **Inferred context (image `csr1.PNG`):** likely a diagram/screenshot showing this exact end-to-end sequence visually — `jane.key`/`jane.csr` generated locally → base64-encoded → pasted into the `CertificateSigningRequest` manifest → `kubectl create -f jane.yaml` submitted to the cluster.

### Managing the CSR

- **To list the CSRs:**
  ```
  $ kubectl get csr
  ```
- **Approve the request:**
  ```
  $ kubectl certificate approve jane
  ```
- **To view the certificate** (once approved and signed):
  ```
  $ kubectl get csr jane -o yaml
  ```
- **To decode it:**
  ```
  $ echo "<certificate>" | base64 --decode
  ```
  - The signed certificate appears (base64-encoded) under the CSR object's `status.certificate` field once approved; `base64 --decode` converts it back into a readable PEM certificate that can be extracted and handed to the requesting user (`jane`) for use in her own kubeconfig.
  - **Inferred context (image `csr2.PNG`):** likely shows this exact `kubectl get csr jane -o yaml` output with the `status.certificate:` field populated, and/or the decoded PEM certificate output, closing the loop on the whole CSR lifecycle: request → submit → approve → extract → decode → deliver to user.

### All certificate-related operations are carried out by the controller manager

- **All the certificate-related operations are carried out by the controller manager.**
- **If anyone has to sign the certificates they need the CA server's root certificate and private key.** The **controller manager** configuration has **two options** where you can specify these.
  - **Inferred context (image `csr3.PNG`):** likely shows the two relevant kube-controller-manager flags — most plausibly **`--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt`** and **`--cluster-signing-key-file=/etc/kubernetes/pki/ca.key`** — pointing the controller-manager at the cluster CA's certificate and private key so it is able to actually perform the signing operation whenever a CSR is approved.
  - **Inferred context (image `csr4.PNG`):** likely shows these two flags located inside the `kube-controller-manager.yaml` static Pod manifest (`/etc/kubernetes/manifests/kube-controller-manager.yaml`) on a kubeadm cluster, confirming exactly where an administrator would look to verify (or troubleshoot) that the controller-manager has the correct CA material configured for signing CSRs.

### K8s Reference Docs

- https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/
- https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/

---

## 12. Practice Test — Certificates API

- Practice test reference: https://kodekloud.com/topic/practice-test-certificates-api/
- Full step-by-step lab-guide walkthrough of the solutions:

1. **"A new member akshay joined our team. He requires access to our cluster. The Certificate Signing Request is at the `/root` location."**
   ```
   $ ls -l /root
   ```
   - Locate the CSR file(s) already prepared for akshay in `/root`.

2. **"View the answer at `/var/answers/akshay-csr.yaml`."**
   ```
   $ kubectl create -f /var/answers/akshay-csr.yaml
   ```
   - Creates the `CertificateSigningRequest` Kubernetes object for akshay (the lab environment provides the pre-built manifest as a reference/answer file).

3. **"Run the command `kubectl get csr`."**
   ```
   $ kubectl get csr
   ```
   - Confirms `akshay`'s CSR now exists and shows its `CONDITION` as `Pending`.

4. **"Run the command `kubectl certificate approve akshay`."**
   ```
   $ kubectl certificate approve akshay
   ```
   - Approves akshay's pending CSR, triggering the controller-manager to sign it using the cluster CA.

5. **"Run the command `kubectl get csr`."**
   ```
   $ kubectl get csr
   ```
   - Confirms akshay's CSR's `CONDITION` now shows `Approved,Issued`.

6. **"Run the command `kubectl get csr` and look at the Requestor column."**
   ```
   $ kubectl get csr
   ```
   - The `REQUESTOR` column identifies who/what submitted each CSR — used here to notice that **other CSRs exist in the list beyond akshay's own request**.

7. **(Information-only step — no command.)** *"The other CSRs are requested during the TLS Bootstrapping process. We will discuss more about it later in the course when we go through the TLS bootstrap section."*
   - Explains why unfamiliar/system-generated CSRs (e.g. from kubelets performing TLS bootstrap when joining the cluster) may legitimately show up alongside user-submitted ones like akshay's — not every CSR in the list is a "new human user" request.

8. **"Run the command `kubectl get csr`."**
   ```
   $ kubectl get csr
   ```
   - Re-inspect the CSR list — this time to notice a specific, suspicious entry named **`agent-smith`** (a deliberately planted "attacker" scenario in the lab).

9. **"Run the command `kubectl get csr agent-smith -o yaml`."**
   ```
   $ kubectl get csr agent-smith -o yaml
   ```
   - Inspect the full YAML of the suspicious `agent-smith` CSR — e.g. checking its requested `usages`/`groups` for signs it's asking for elevated/inappropriate privileges (the classic "Matrix" naming joke signaling this is meant to be treated as a rogue/untrusted request).

10. **"Run the command `kubectl certificate deny agent-smith`."**
    ```
    $ kubectl certificate deny agent-smith
    ```
    - **Denies** the suspicious CSR rather than approving it — the correct security response to an untrusted/unexpected certificate request.

11. **"Run the command `kubectl delete csr agent-smith`."**
    ```
    $ kubectl delete csr agent-smith
    ```
    - Cleans up by deleting the denied CSR object entirely from the cluster.

**Exam-relevant takeaway:** the full lifecycle exercised here is `kubectl create -f <csr>.yaml` → `kubectl get csr` (check `CONDITION`/`REQUESTOR`) → `kubectl certificate approve <name>` **or** `kubectl certificate deny <name>` → optionally `kubectl delete csr <name>`. Not every CSR you see in `kubectl get csr` is a legitimate new-user request — some are automatically generated during **kubelet TLS bootstrapping**, and a CSR you don't recognize or that requests suspicious permissions should be **denied**, not approved, then deleted.

---

## 13. KubeConfig

- Video reference: https://kodekloud.com/topic/kubeconfig/
- Topic: rather than passing certificate/key/server flags manually on every single `kubectl`/`curl` call, Kubernetes lets you store that connection information in a **kubeconfig** file.

### From raw curl/kubectl flags to a kubeconfig file

- **A client uses the certificate file and key to query the kubernetes REST API for a list of pods, using `curl`.** You can specify the same information using `kubectl` (equivalent flags: `--server`, `--client-certificate`, `--client-key`, `--certificate-authority`).
  - **Inferred context (image `kc1.PNG`):** likely shows a side-by-side comparison of a raw `curl` invocation (e.g. `curl https://master-node-ip:6443/api/v1/pods --key admin.key --cert admin.crt --cacert ca.crt`) against the equivalent `kubectl` command (`kubectl get pods --server https://master-node-ip:6443 --client-key admin.key --client-certificate admin.crt --certificate-authority ca.crt`) — motivating the kubeconfig file as a way to avoid retyping all of these flags on every command.
- **We can move this information into a configuration file called `kubeconfig`**, and then specify this file via the `--kubeconfig` option:
  ```
  $ kubectl get pods --kubeconfig config
  ```

### Kubeconfig File structure

- **The kubeconfig file has 3 sections:**
  - **Clusters**
  - **Contexts**
  - **Users**
- **Inferred context (image `kc4.PNG`):** likely a diagram explaining each of the 3 sections' purpose:
  - **Clusters** — the set of Kubernetes clusters this kubeconfig knows how to reach (each entry has a `name`, the cluster's API `server` URL, and its `certificate-authority`/`certificate-authority-data`).
  - **Users** — the set of user identities this kubeconfig can authenticate as (each entry has a `name` and its credentials — typically `client-certificate`/`client-certificate-data` + `client-key`/`client-key-data`, or a token).
  - **Contexts** — the glue that ties a **cluster** + a **user** (+ optionally a default **namespace**) together under a friendly name, so that switching "context" switches which cluster you're pointed at *and* which user identity you're authenticating as, together, in one step.
- **Inferred context (image `kc5.PNG`):** likely shows a concrete sample kubeconfig YAML file with all 3 sections populated (something structurally equivalent to):
  ```yaml
  apiVersion: v1
  kind: Config
  clusters:
  - name: my-kube-playground
    cluster:
      certificate-authority: ca.crt
      server: https://my-kube-playground:6443
  contexts:
  - name: my-kube-admin@my-kube-playground
    context:
      cluster: my-kube-playground
      user: my-kube-admin
  users:
  - name: my-kube-admin
    user:
      client-certificate: admin.crt
      client-key: admin.key
  current-context: my-kube-admin@my-kube-playground
  ```
  - **Naming convention called out:** context names are conventionally written as `<user>@<cluster>` (e.g. `prod-user@production`), which is exactly the format used moments later in the `use-context` example below.
  - By default, `kubectl` looks for this file at **`$HOME/.kube/config`**.

### Viewing and switching kubeconfig context

- **To view the current file being used:**
  ```
  $ kubectl config view
  ```
- **You can specify the kubeconfig file with `kubectl config view` using the `--kubeconfig` flag:**
  ```
  $ kubectl config veiw --kubeconfig=my-custom-config
  ```
  *(Note: the source text has a typo, `veiw` instead of `view` — the correct command is `kubectl config view --kubeconfig=my-custom-config`.)*
  - **Inferred context (image `kc6.PNG`):** likely shows sample `kubectl config view` output — the same 3-section (`clusters:`/`contexts:`/`users:`) YAML structure printed to the terminal, plus a `current-context:` field at the bottom.
- **How do you update your current context? (change the active cluster+user pairing):**
  ```
  $ kubectl config use-context <context-name>
  ex:
  $ kubectl config use-context prod-user@production
  ```
  - **Inferred context (image `kc7.PNG`):** likely shows the `current-context:` field in `kubectl config view`'s output changing before/after running `use-context`, confirming the switch took effect.
- **`kubectl config` help:**
  ```
  $ kubectl config -h
  ```
  - **Inferred context (image `kc8.PNG`):** likely shows the help output listing all `kubectl config` subcommands (`view`, `use-context`, `current-context`, `get-contexts`, `set-context`, `set-cluster`, `set-credentials`, `delete-context`, `rename-context`, etc.), giving a fuller picture of what can be managed beyond just viewing/switching.

### What about namespaces?

- **Inferred context (image `kc9.PNG`):** likely explains that a **context** can optionally bind to a specific **default namespace** (via a `namespace:` field under `context:` in the kubeconfig), so that once that context is active, `kubectl` commands run against that namespace by default without needing `-n <namespace>` on every command — useful for admins who regularly work inside one particular namespace (e.g. a `dev` team context defaulting into the `dev` namespace).

### Certificates in kubeconfig

- **Inferred context (image `kc10.PNG`):** likely explains the two ways certificate material can be referenced inside a kubeconfig file: either as a **file path** (e.g. `certificate-authority: /etc/kubernetes/pki/ca.crt`, `client-certificate: /path/to/admin.crt`) pointing to a `.crt`/`.pem`/`.key` file on disk, **or** embedded directly as **base64-encoded data** inline in the file, using the `-data` suffixed keys instead:
  - `certificate-authority-data:` (instead of `certificate-authority:`)
  - `client-certificate-data:` (instead of `client-certificate:`)
  - `client-key-data:` (instead of `client-key:`)
- **Inferred context (image `kc12.PNG`):** likely shows a concrete example of the **embedded/base64** form in a kubeconfig YAML, e.g.:
  ```yaml
  users:
  - name: my-kube-admin
    user:
      client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FU...
      client-key-data: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS...
  ```
- **Inferred context (image `kc11.PNG`):** likely shows the alternative **file-path** form side by side for direct comparison, and/or explains the practical trade-off: file-path references keep the kubeconfig file small and human-readable but require the referenced cert/key files to actually exist at that path on whatever machine the kubeconfig is used from, whereas the `-data` embedded form makes the kubeconfig fully **self-contained and portable** (no external files needed) at the cost of a much larger, less readable file — which is exactly the failure mode exercised in file 14's practice test, where a `client-certificate` file-path entry points at the wrong location and must be corrected.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/
- https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config

---

## 14. Practice Test — KubeConfig

- Practice test reference: https://kodekloud.com/topic/practice-test-kubeconfig/
- Full step-by-step lab-guide walkthrough of the solutions:

1. **"Look for the kube config file under `/root/.kube`."**
   ```
   $ ls -l /root/.kube
   ```
   - Confirms the default kubeconfig file (`config`) exists at the default location `$HOME/.kube/config`.

2. **"Run the `kubectl config view` command and count the number of clusters."**
   ```
   $ kubectl config view
   ```
   - Count the entries under the `clusters:` section of the output.

3. **"Run the command `kubectl config view` and count the number of users."**
   ```
   $ kubectl config view
   ```
   - Count the entries under the `users:` section of the output.

4. **"How many contexts are defined in the default kubeconfig file?"**
   ```
   $ kubectl config view
   ```
   - Count the entries under the `contexts:` section of the output.

5. **"Run the command `kubectl config view` and look for the user name."**
   ```
   $ kubectl config view
   ```
   - Read the `name:` field(s) under `users:`.

6. **"What is the name of the cluster configured in the default kubeconfig file?"**
   ```
   $ kubectl config view
   ```
   - Read the `name:` field under `clusters:`.

7. **"Run the command `kubectl config view --kubeconfig my-kube-config`."**
   ```
   $ kubectl config view --kubeconfig my-kube-config
   ```
   - Switches focus to inspecting a **separate**, non-default kubeconfig file (`my-kube-config`), demonstrating the `--kubeconfig` flag introduced in file 13 to view any arbitrary kubeconfig file, not just the default `~/.kube/config`.

8. **"How many contexts are configured in the 'my-kube-config' file?"**
   ```
   $ kubectl config view --kubeconfig my-kube-config
   ```
   - Count the entries under `contexts:` in this alternate file's output.

9. **"What user is configured in the 'research' context?"**
   ```
   $ kubectl config view --kubeconfig my-kube-config
   ```
   - Find the `contexts:` entry named `research`, and read its `context.user:` field.

10. **"What is the name of the client-certificate file configured for the 'aws-user'?"**
    ```
    $ kubectl config view --kubeconfig my-kube-config
    ```
    - Find the `users:` entry named `aws-user`, and read its `user.client-certificate:` field (the file-path form of certificate reference, per file 13's discussion of file-path vs. `-data` embedded certs).

11. **"What is the current context set to in the 'my-kube-config' file?"**
    ```
    $ kubectl config view --kubeconfig my-kube-config
    ```
    - Read the `current-context:` field at the bottom of the output.

12. **"Run the command `kubectl config --kubeconfig=/root/my-kube-config use-context research`."**
    ```
    $ kubectl config --kubeconfig=/root/my-kube-config use-context research
    ```
    - Switches the **current context** inside the `my-kube-config` file itself to `research` — note the `--kubeconfig` flag must be given *before* the `use-context` subcommand acts on that specific file, otherwise `use-context` would modify the default `~/.kube/config` instead.

13. **"Replace the contents in the default kubeconfig file with the content from `my-kube-config` file."**
    ```
    $ mv .kube/config .kube/config.bak
    $ cp /root/my-kube-config .kube/config
    ```
    - First **backs up** the existing default config (`config` → `config.bak`, so nothing is destructively lost), then **copies** `my-kube-config`'s contents over to become the new default `~/.kube/config` — after this, plain `kubectl` commands (with no `--kubeconfig` flag at all) will use what used to be `my-kube-config`'s clusters/contexts/users.

14. **"The path to certificate is incorrect in the kubeconfig file. Fix it. All users' certificates are stored at `/etc/kubernetes/pki/users`."**
    - **Diagnose the problem** — running a normal command fails or misbehaves because of the bad path:
      ```
      $ kubectl get pods
      ```
    - **Locate the actual certificate files** on disk to find their real, correct path:
      ```
      master $ ls
      dev-user.crt  dev-user.csr  dev-user.key
      ```
    - **Open the kubeconfig file to edit it:**
      ```
      master $ vi /root/.kube/config
      ```
    - **Confirm the (incorrect) path currently configured**, by grepping the file:
      ```
      master $ grep dev-user.crt /root/.kube/config
        client-certificate: /etc/kubernetes/pki/users/dev-user/dev-user.crt
      ```
    - **Check where you actually are / where the files actually live**, to compare against the path found above:
      ```
      master $ pwd
      /etc/kubernetes/pki/users/dev-user
      ```
      - This confirms the *directory* portion of the `client-certificate:` path is in fact correct (`/etc/kubernetes/pki/users/dev-user/`) — meaning the actual bug is more subtle than a wholesale wrong directory (e.g. a wrong filename, a typo, or a mismatched `client-key` path elsewhere in the same `users:` entry that needs the same fix applied) — the fix is to edit the `vi` session opened above so that **both** the `client-certificate:` and `client-key:` fields under the relevant `users:` entry correctly resolve to the real `dev-user.crt` / `dev-user.key` files just confirmed to exist at that path.
    - **Verify the fix:**
      ```
      master $ kubectl get pods
      No resources found in default namespace.
      ```
      - The command now **succeeds** (authenticating fine) rather than erroring out — `No resources found in default namespace.` is the expected, healthy response when there simply are no Pods in that namespace, proving the certificate path issue is resolved (the command reached and was authenticated by the API server rather than failing on a TLS/file-not-found error).

**Exam-relevant takeaway:** `kubectl config view [--kubeconfig <file>]` is the universal tool for reading any kubeconfig's `clusters:`/`contexts:`/`users:` sections and its `current-context:`; `kubectl config [--kubeconfig <file>] use-context <name>` switches which context is active *within that specific file*. When told a "path to a certificate is incorrect," the fix is always the same pattern: find where the real cert/key files actually live (`ls`/`pwd`), then edit the kubeconfig's `client-certificate`/`client-key` (or `certificate-authority`) fields to match — and always sanity-check the fix afterward with a plain `kubectl get pods` (or similar) to confirm the request now authenticates successfully.

---

## 15. API Groups

- Video reference: https://kodekloud.com/topic/api-groups/
- Topic: the structure of the Kubernetes REST API itself — how its various endpoints are organized into logical "groups," and the practical implications for authenticating against it.

### Returning version info and listing pods via the API

- **Inferred context (image `api3.PNG`):** likely shows two raw HTTPS calls directly against the kube-apiserver (e.g. via `curl` with cert/key/cacert flags, or through `kubectl proxy`) — one hitting `/version` to return the cluster's Kubernetes version info, and one hitting `/api/v1/pods` to return the list of all Pods across the cluster (or in a namespace) — establishing that the Kubernetes API is just a REST API you can talk to directly, not something exclusively mediated by `kubectl`.

### The API is grouped into multiple groups based on purpose

- **The kubernetes API is grouped into multiple such groups based on their purpose**, such as one for **`APIs`**, one for **`healthz`**, **`metrics`**, and **`logs`**, etc.
- **Inferred context (image `api4.PNG`):** likely a tree/list diagram of the top-level API paths exposed by the apiserver, most plausibly including:
  - `/api` and `/apis` — the actual resource APIs (Pods, Deployments, Services, etc.).
  - `/metrics` — Prometheus-format metrics about the apiserver itself.
  - `/healthz` — health/liveness check endpoint(s).
  - `/logs` — access to apiserver logs.
  - `/version` — version info (as queried in `api3.PNG` above).

### API and APIs — the two functional categories

- These functional resource APIs are categorized into two:
  - **The core group** (exposed under **`/api`**) — **where all the [original/foundational] functionality exists**: Pods, Namespaces, ReplicationControllers, Events, Endpoints, Nodes, Bindings, PersistentVolumes/Claims, ConfigMaps, Secrets, Services, etc.
    - **Inferred context (image `api5.PNG`):** likely shows the `/api/v1` endpoint's structure/response listing these core resource types directly under the single version `v1` (no separate sub-group name), consistent with "core" resources having existed since the very earliest versions of Kubernetes and never having been reorganized into a named group.
  - **The Named group** (exposed under **`/apis/<group-name>/<version>`**) — **more organized, and going forward all the newer features are going to be made available [under] these named groups.** Examples: `apps` (Deployments, ReplicaSets, StatefulSets, DaemonSets), `certificates.k8s.io` (CertificateSigningRequest — as seen in file 11), `extensions`, `networking.k8s.io` (NetworkPolicy, Ingress), `storage.k8s.io`, `authorization.k8s.io`, `rbac.authorization.k8s.io`, etc.
    - **Inferred context (image `api6.PNG`):** likely shows the `/apis` endpoint's response — a list of every named group (`apps`, `networking.k8s.io`, `rbac.authorization.k8s.io`, `storage.k8s.io`, etc.), each further broken down by its own supported version(s) (e.g. `apps/v1`), illustrating the more structured, extensible organization that all new Kubernetes features are added into going forward, versus the flat, frozen `core`/`v1` group.
- **To list all the API groups:**
  - **Inferred context (image `api7.PNG`):** likely shows the actual command/output used to enumerate them — most plausibly `curl https://localhost:6443/apis -k --key admin.key --cert admin.crt --cacert ca.crt | jq .` (or the equivalent unauthenticated-but-cert-authenticated call), returning a JSON `APIGroupList` document listing every named group and its versions.

### Note on accessing the kube-apiserver

- **You have to authenticate by passing the certificate files** — i.e. direct calls against the apiserver's HTTPS endpoint require presenting a valid client certificate (or other credential) exactly as covered in files 03–14.
- **Inferred context (image `api8.PNG`):** likely shows a `curl` example explicitly passing `--key`, `--cert`, and `--cacert` flags against the apiserver's HTTPS port (e.g. `https://master-node-ip:6443/api/v1/pods`), reinforcing that raw API access needs the same certificate material a kubeconfig file would otherwise supply automatically.
- **An alternative is to start a `kubeproxy` client** — i.e. run **`kubectl proxy`**, which starts a local proxy server that handles authentication using your existing kubeconfig credentials, and then exposes the Kubernetes API **unauthenticated** on localhost, so subsequent `curl` calls to `http://localhost:8001/...` need no certificate flags at all (the proxy has already done the authenticating on your behalf, using your kubeconfig).
- **Inferred context (image `api9.PNG`):** likely shows exactly this: `kubectl proxy` started in one terminal (default listening on `127.0.0.1:8001`), followed by a plain `curl http://localhost:8001/api/v1/pods` (no cert flags needed) in another terminal, succeeding — visually demonstrating the convenience this "proxy" approach provides over raw cert-based `curl` calls.

### `kube-proxy` vs `kubectl proxy`

- **Inferred context (image `kp.PNG`):** these two tools are commonly confused by name alone, so this is almost certainly a disambiguation table:
  - **`kube-proxy`** — a **cluster networking component** that runs on every node, responsible for implementing Kubernetes **Services** (maintaining `iptables`/`ipvs` rules that route Service ClusterIPs to backing Pod IPs). It has nothing to do with accessing the API server.
  - **`kubectl proxy`** — a **client-side convenience command** (as just described above) that creates a local authenticated HTTP proxy to the Kubernetes API server, for easy unauthenticated `curl`/browser access to the API on `localhost`.
  - **Exam-relevant:** these names sound almost identical but serve entirely unrelated purposes — `kube-proxy` is a core cluster component that must always be running for Services to work; `kubectl proxy` is an optional, on-demand debugging/access convenience tool a user runs locally.

### Key Takeaways

- **Inferred context (image `api10.PNG`):** likely a summary slide recapping this file's core points:
  - The Kubernetes API is organized into the **core group** (`/api/v1` — original/foundational resources) and multiple **named groups** (`/apis/<group>/<version>` — all newer features going forward).
  - You can talk to the apiserver **directly** via any HTTPS client (`curl`, a browser, custom code) as long as you supply valid **authentication** (typically client certificates).
  - **`kubectl proxy`** is a convenient way to get authenticated, cert-free `localhost` access to the full API for quick testing/scripting.
  - Don't confuse **`kubectl proxy`** (API access helper) with **`kube-proxy`** (the cluster's Service-networking component) — same word, completely different roles.

### K8s Reference Docs

- https://kubernetes.io/docs/concepts/overview/kubernetes-api/
- https://kubernetes.io/docs/reference/using-api/api-concepts/
- https://kubernetes.io/docs/tasks/extend-kubernetes/http-proxy-access-api/

---

*Continued in [kubernetes-security-notes_part_03.md](kubernetes-security-notes_part_03.md)
— Sections 16–22 (Authorization, RBAC + Practice Test, Cluster Roles + Practice Test,
Service Accounts + Practice Test).*
