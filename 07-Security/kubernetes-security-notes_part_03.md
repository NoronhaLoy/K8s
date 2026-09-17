# Security — Complete Notes (Part 3 of 4)

> **Part 3 of 4** — covers Sections 16–22 (Authorization, RBAC + Practice Test, Cluster
> Roles + Practice Test, Service Accounts + Practice Test).
> Previous: [kubernetes-security-notes_part_02.md](kubernetes-security-notes_part_02.md)
> (Sections 09–15).
> Next: [kubernetes-security-notes_part_04.md](kubernetes-security-notes_part_04.md)
> (Sections 23–30 + Quick Revision Checklist).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/07-Security/`

---

## 16. Authorization

- Video reference: *Authorization* — https://kodekloud.com/topic/authorization/
- Topic: once a user is **authenticated** (proven who they are), Kubernetes still needs to decide **what they are allowed to do** — that's the job of **Authorization**.

### Why do you need Authorization in your cluster?

- As an **admin**, you can perform all operations without restriction:
  ```
  $ kubectl get nodes
  $ kubectl get pods
  $ kubectl delete node worker-2
  ```
- But not every user of the cluster should be able to run every command (e.g. a developer shouldn't necessarily be able to delete a node) — this is the motivating problem that authorization mechanisms solve.
  - **Inferred context (image `at1.PNG`):** likely illustrates two different users hitting the API server with the same kind of request (e.g. `kubectl delete node`) — one is an admin (allowed) and one is a regular/dev user (denied) — visually motivating why some layer beyond authentication must gate what an already-authenticated identity can *do*.

### Authorization Mechanisms

- There are different authorization mechanisms supported by Kubernetes:
  - **Node Authorization**
  - **Attribute-based Authorization (ABAC)**
  - **Role-Based Authorization (RBAC)**
  - **Webhook**

### Node Authorization

- **Inferred context (image `node-auth.png`):** Node Authorization is a special-purpose authorizer built specifically for the **kubelet's** API requests. It most likely shows the kubelet (identified via the `system:node:<nodeName>` username and `system:nodes` group, part of the "Node Identity" established during TLS bootstrapping) sending requests to the kube-apiserver (e.g. to read/update the status of Pods, Nodes, Services, endpoints, and Secrets/ConfigMaps bound to Pods on that node), with the Node Authorizer permitting only those requests a kubelet legitimately needs to make about its own node and the Pods scheduled to it.

### ABAC

- **Inferred context (image `abac.PNG`):** Attribute-Based Access Control associates a user (or group) with a set of permissions via a **policy file** made up of individual JSON lines (policy objects), each attribute-matching a user to allowed actions/resources — e.g. `{"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup": "*"}}`. The diagram likely shows this raw JSON policy file being read directly by the API server at startup. The well-known downside (general Kubernetes knowledge underlying why RBAC superseded ABAC) is that policies must be edited manually as text files and the API server restarted to pick up changes — hard to manage/audit at scale.

### RBAC

- **Inferred context (image `rbac.PNG`):** Role-Based Access Control associates a user (or group of users) with a set of permissions defined declaratively via **Role** objects (a set of rules on resources/verbs), then **binds** users to those roles with **RoleBinding** objects — rather than a static attribute policy file, permissions are managed as first-class Kubernetes API objects that can be created/updated/queried with `kubectl` like any other resource. This is expanded in full in file 17.

### Webhook

- **Inferred context (image `webhook.PNG`):** Webhook mode delegates the authorization decision to an **external service** — for every request, the API server calls out (over HTTP) to a third-party authorization service (the diagram's classic example in Kubernetes docs is integrating with **Open Policy Agent (OPA)**), passing the request's attributes, and the external service returns an allow/deny decision back to the API server. This lets organizations plug in centralized, custom, or off-the-shelf policy engines instead of relying on Kubernetes' built-in modes.

### Authorization Modes

- The mode options can be defined on the **kube-apiserver** (via the `--authorization-mode` flag).
  - **Inferred context (image `mode.PNG`):** likely shows the `--authorization-mode` flag on the `kube-apiserver` process/manifest with a comma-separated list of values, e.g. `--authorization-mode=Node,RBAC,Webhook`, plus the special value **`AlwaysAllow`** (default if unset — allows all requests, i.e. no authorization) and **`AlwaysDeny`** (denies all requests) that also exist as valid modes.
- When you specify **multiple modes**, Kubernetes authorizes the request **in the order in which the modes are specified**.
  - **Inferred context (image `mode1.PNG`):** likely shows the chained evaluation flow — e.g. with `--authorization-mode=Node,RBAC,Webhook`, a request is first checked against the **Node** authorizer; if that authorizer doesn't have an opinion (doesn't explicitly allow it), the request moves on to be checked by **RBAC**; if RBAC also has no opinion, it finally falls through to **Webhook**. **The request is authorized as soon as any one authorizer in the chain approves it** — if none of the configured authorizers approve the request, it is denied.

### K8s Reference Docs

- https://kubernetes.io/docs/reference/access-authn-authz/authorization/

---

## 17. RBAC

- Video reference: *Role Based Access Controls* — https://kodekloud.com/topic/role-based-access-controls/

### How do we create a role?

- Each **Role** has 3 sections per rule:
  - `apiGroups`
  - `resources`
  - `verbs`
- Create the role with the `kubectl` command:
  ```
  $ kubectl create -f developer-role.yaml
  ```

### The next step is to link the user to that role

- To associate a user with a Role, create another object called a **`RoleBinding`**. This RoleBinding object links a user object to a role.
- Create the role binding using the `kubectl` command:
  ```
  $ kubectl create -f devuser-developer-binding.yaml
  ```
- **Roles and RoleBindings fall under the scope of a namespace** (they are namespaced objects).
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: developer
  rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["pods"]
    verbs: ["get", "list", "update", "delete", "create"]
  - apiGroups: [""]
    resources: ["ConfigMap"]
    verbs: ["create"]
  ```
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: devuser-developer-binding
  subjects:
  - kind: User
    name: dev-user # "name" is case sensitive
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: developer
    apiGroup: rbac.authorization.k8s.io
  ```
  - **Inferred context (image `rbac1.PNG`):** likely a diagram showing the relationship visually: `[dev-user] --(subjects)--> [RoleBinding: devuser-developer-binding] --(roleRef)--> [Role: developer] --(rules)--> [pods, ConfigMap resources with get/list/update/delete/create/create verbs]` — the RoleBinding acting as the glue/edge connecting a subject (user) to a Role's permission set, all scoped to one namespace.

### View RBAC

- To list roles:
  ```
  $ kubectl get roles
  ```
- To list rolebindings:
  ```
  $ kubectl get rolebindings
  ```
- To describe a role:
  ```
  $ kubectl describe role developer
  ```
  - **Inferred context (image `rbac2.PNG`):** likely sample `kubectl describe role developer` output — a table of `PolicyRule` entries listing `Resources | Non-Resource URLs | Resource Names | Verbs`, matching the `apiGroups`/`resources`/`verbs` defined in the YAML above.
- To describe a rolebinding:
  ```
  $ kubectl describe rolebinding devuser-developer-binding
  ```
  - **Inferred context (image `rbac3.PNG`):** likely sample `kubectl describe rolebinding devuser-developer-binding` output — showing the `Role` it references (`Kind: Role, Name: developer`) and the `Subjects` table (`Kind: User, Name: dev-user, Namespace: ...`).

### Check Access

- What if you, as a user, would like to see if you have access to a particular resource in the cluster? Use the `kubectl auth` command:
  ```
  $ kubectl auth can-i create deployments
  $ kubectl auth can-i delete nodes
  ```
  ```
  $ kubectl auth can-i create deployments --as dev-user
  $ kubectl auth can-i create pods --as dev-user
  ```
  ```
  $ kubectl auth can-i create pods --as dev-user --namespace test
  ```
  - `--as <user>` lets an admin **impersonate** another user to check what *they* can do, without needing that user's actual credentials.
  - `--namespace <ns>` scopes the check to a specific namespace (relevant since Roles/RoleBindings are namespaced).
  - **Inferred context (image `rbac5.PNG`):** likely shows sample terminal output of these `can-i` commands, each simply printing `yes` or `no` depending on whether the (impersonated) user's RBAC permissions allow the given verb/resource/namespace combination.

### Resource Names

- Note: you can restrict a Role's permissions down to **specific named instances** of a resource using `resourceNames`, rather than granting access to *all* pods/objects of that resource type in the namespace.
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: developer
  rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["pods"]
    verbs: ["get", "update", "create"]
    resourceNames: ["blue", "orange"]
  ```
  - **Inferred context (image `rbac4.PNG`):** likely visualizes this restriction — e.g. showing three Pods named `blue`, `orange`, and `green` in a namespace, with the Role's `get`/`update` access permitted only against `blue` and `orange`, while `green` remains inaccessible under this rule (note: `resourceNames` typically cannot be combined with the `create` verb meaningfully, since a not-yet-created object has no name to match against — a subtlety worth remembering for the exam even though the YAML above lists `create` alongside `resourceNames`).

### K8s Reference Docs

- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/#command-line-utilities

---

## 18. Practice Test — RBAC

Solutions to practice test — RBAC. Step-by-step lab-guide walkthrough:

1. **"Run the command `kubectl describe pod kube-apiserver-controlplane -n kube-system` and look for `--authorization-mode`."**
   ```
   $ kubectl describe pod kube-apiserver-controlplane -n kube-system
   ```
   - Inspect the `Command`/args section of the described Pod for the `--authorization-mode` flag — its value (a comma-separated list, e.g. `Node,RBAC`) reveals which authorization modes are active on this cluster, and confirms whether RBAC is enabled at all.

2. **"Run the command `kubectl get roles`."**
   ```
   $ kubectl get roles
   ```
   - Lists Roles in the **current namespace only** (Roles are namespaced objects).

3. **"Run the command `kubectl get roles --all-namespaces`."**
   ```
   $ kubectl get roles --all-namespaces
   ```
   - Lists Roles across every namespace in the cluster, useful for finding system-created Roles like `kube-proxy`'s.

4. **"Run the command `kubectl describe role kube-proxy -n kube-system`."**
   ```
   $ kubectl describe role kube-proxy -n kube-system
   ```
   - Shows the `kube-proxy` Role's rules — its `apiGroups`/`resources`/`verbs`/`resourceNames`.

5. **"Check the verbs associated to the kube-proxy role."**
   ```
   $ kubectl describe role kube-proxy -n kube-system
   ```
   - Read the `Verbs` column of the printed `PolicyRule` table.

6. **"Which of the following statements are true?"**
   - **Answer:** *kube-proxy role can get details of configmap object by the name kube-proxy.*
   - This follows directly from reading the Role's rule: it grants `get` on `configmaps` restricted (via `resourceNames`) to the specific object named `kube-proxy` — not all ConfigMaps in the namespace.

7. **"Run the command `kubectl describe rolebinding kube-proxy -n kube-system`."**
   ```
   $ kubectl describe rolebinding kube-proxy -n kube-system
   ```
   - Confirms which subject (user/group/service account — for `kube-proxy` this is typically the `system:bootstrappers` group / node-related identity) is bound to the `kube-proxy` Role.

8. **"Run the command `kubectl get pods --as dev-user`."**
   ```
   $ kubectl get pods --as dev-user
   ```
   - Impersonates `dev-user` to test whether that user (as currently configured via RBAC) is authorized to list Pods — will error with a `Forbidden` message if `dev-user` has no Role/RoleBinding granting `get`/`list` on pods.

9. **"Answer file located at `/var/answers`."** (task: create a Role granting `dev-user` the needed permissions)
   ```
   $ kubectl create -f /var/answers/developer-role.yaml
   ```
   - Applies the pre-written answer manifest that defines the correct `developer` Role (and, implicitly, its binding) satisfying the task's requirements.

10. **"New roles and role bindings are created in the `blue` namespace. Check it out. Check the `resourceNames` configured on the role."**
    ```
    $ kubectl get roles,rolebindings -n blue
    $ kubectl describe role developer -n blue
    $ kubectl edit role developer -n blue   # (update the resourceNames)
    ```
    - First inspect the existing Role/RoleBinding in the `blue` namespace, note the current `resourceNames` restriction, then edit the Role in place to update the list of allowed resource names to match the task's requirement.

11. **"View the answer file located at `/var/answers/dev-user-deploy.yaml`."**
    ```
    $ kubectl create -f /var/answers/dev-user-deploy.yaml
    ```
    - Applies a final pre-written answer manifest (e.g. covering Deployment-related permissions for `dev-user`) to complete the lab.

**Exam-relevant takeaway:** always check `--authorization-mode` on the kube-apiserver first to confirm RBAC is active; use `--all-namespaces` since Roles/RoleBindings are namespace-scoped and easy to miss if you only check the "current" namespace; `kubectl describe role`/`kubectl describe rolebinding` are the two commands to fully read out a permission set and who it's bound to; `kubectl get pods --as <user>` (or `kubectl auth can-i ... --as <user>`) is the standard way to verify RBAC configuration from another user's perspective without switching kubeconfig contexts.

---

## 19. Cluster Roles

- Video reference: *Cluster Roles* — https://kodekloud.com/topic/cluster-roles/

### Roles

- **Roles and RoleBindings are namespaced**, meaning they are created within (and only apply within) a namespace.
  - **Inferred context (image `roles.PNG`):** likely shows a Role/RoleBinding pair drawn *inside* a namespace boundary box, visually reinforcing that their effect stops at the namespace edge — a user bound to a Role in namespace `A` has no permissions in namespace `B` even for the identical resource type.

### Namespaces

- **Can you group or isolate nodes within a namespace?**
  - **No** — Nodes are **cluster-wide / cluster-scoped resources**. They cannot be associated with any particular namespace.
  - **Inferred context (image `namespace.PNG`):** likely shows a Node object sitting *outside*/above any namespace boxes, illustrating that cluster-scoped resources exist independently of the namespace partitioning that applies to objects like Pods, Deployments, Roles, etc.
- So resources in Kubernetes are categorized as either **namespaced** or **cluster-scoped**.
- To see namespaced resources:
  ```
  $ kubectl api-resources --namespaced=true
  ```
- To see non-namespaced resources:
  ```
  $ kubectl api-resources --namespaced=false
  ```
  - **Inferred context (image `namespace1.PNG`):** likely shows sample output of `kubectl api-resources --namespaced=false` — a table listing cluster-scoped resource kinds such as `nodes`, `namespaces` themselves, `persistentvolumes`, `clusterroles`, `clusterrolebindings`, `certificatesigningrequests`, and `storageclasses`.

### Cluster Roles and Cluster Role Bindings

- **ClusterRoles** are like Roles, except they apply to **cluster-scoped resources**. Kind is **`ClusterRole`**.
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: cluster-administrator
  rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["nodes"]
    verbs: ["get", "list", "delete", "create"]
  ```
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: cluster-admin-role-binding
  subjects:
  - kind: User
    name: cluster-admin
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: cluster-administrator
    apiGroup: rbac.authorization.k8s.io
  ```
  ```
  $ kubectl create -f cluster-admin-role.yaml
  $ kubectl create -f cluster-admin-role-binding.yaml
  ```
  - **Inferred context (image `cr1.PNG`):** likely a diagram parallel to `rbac1.PNG` from file 17, but drawn **outside** any namespace boundary — `[cluster-admin user] --(subjects)--> [ClusterRoleBinding: cluster-admin-role-binding] --(roleRef)--> [ClusterRole: cluster-administrator] --(rules)--> [nodes: get/list/delete/create]` — emphasizing the cluster-wide (not per-namespace) scope of both objects.
- **You can also create a ClusterRole for namespaced resources** (e.g. Pods, Deployments) — when you do this, the user is granted access to those resources **across all namespaces** in the cluster, rather than being confined to one namespace as a normal Role/RoleBinding pair would be. This is the standard technique for granting "read pods everywhere" / cluster-wide administrator-style permissions over namespaced object types.

### K8s Reference Docs

- https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/#command-line-utilities

---

## 20. Practice Test — Cluster Roles

Solutions to practice test — Cluster Roles. Step-by-step lab-guide walkthrough:

1. **"Run the command `kubectl get clusterroles --no-headers | wc -l` or `kubectl get clusterroles --no-headers -o json | jq '.items | length'`."**
   ```
   $ kubectl get clusterroles --no-headers | wc -l (or)
   $ kubectl get clusterroles --no-headers -o json | jq '.items | length'
   ```
   - Counts existing ClusterRoles (system-created + any custom ones) — `--no-headers` strips the header row so `wc -l` gives an exact count; the `jq` variant does the equivalent by counting JSON array items.

2. **"Run the command `kubectl get clusterrolebindings --no-headers | wc -l` or `kubectl get clusterrolebindings --no-headers -o json | jq '.items | length'`."**
   ```
   $ kubectl get clusterrolebindings --no-headers | wc -l (or)
   $ kubectl get clusterrolebindings --no-headers -o json | jq '.items | length'
   ```
   - Same counting technique applied to ClusterRoleBindings.

3. **"What namespace is the `cluster-admin` clusterrole part of?"**
   - **Answer:** *Cluster roles are cluster wide and not part of any namespace.*
   - This is the core conceptual check of the whole lesson — ClusterRoles (and ClusterRoleBindings) are **not** namespaced objects at all, so the question's premise ("what namespace") is itself the trick.

4. **"Run the command `kubectl describe clusterrolebinding cluster-admin`."**
   ```
   $ kubectl describe clusterrolebinding cluster-admin
   ```
   - Shows which subjects (typically the `system:masters` group) are bound to the built-in `cluster-admin` ClusterRole.

5. **"Run the command `kubectl describe clusterrole cluster-admin`."**
   ```
   $ kubectl describe clusterrole cluster-admin
   ```
   - Shows the `cluster-admin` built-in ClusterRole's rules — typically a single rule granting `*` verbs on `*` resources in `*` apiGroups (i.e. unrestricted superuser access), plus `*` on non-resource URLs.

6. **"Check answer at `/var/answers`."** (task: grant `michelle` node-admin-style cluster-wide permissions)
   ```
   $ kubectl create -f /var/answers/michelle-node-admin.yaml
   ```
   - Applies the pre-written answer manifest defining the appropriate ClusterRole + ClusterRoleBinding for user `michelle` over Node resources.

7. **"Check answer at `/var/answers`."** (task: grant `michelle` storage-admin-style cluster-wide permissions)
   ```
   $ kubectl create -f /var/answers/michelle-storage-admin.yaml
   ```
   - Applies a second pre-written answer manifest defining a ClusterRole + ClusterRoleBinding for `michelle` over storage-related cluster-scoped resources (e.g. `persistentvolumes`, `storageclasses`).

**Exam-relevant takeaway:** ClusterRoles/ClusterRoleBindings are always cluster-scoped — never ask/answer "which namespace" for them. `kubectl describe clusterrole`/`clusterrolebinding` are the go-to inspection commands, exactly mirroring the namespaced `describe role`/`rolebinding` commands from file 18. The built-in `cluster-admin` ClusterRole is the canonical "full access to everything" role, typically bound to the `system:masters` group.

---

## 21. Service Account

- Video reference: *Service Account* — https://kodekloud.com/topic/service-account/

### Why Do You Need a Service Account in Your Cluster?

- Kubernetes operates with two main types of users:
  1. **Human Users** — interact with the cluster through tools like `kubectl`.
  2. **Bots/Applications** — services like Prometheus, Jenkins, and other system applications.
- **Service accounts** are primarily used to authenticate these bots or applications to the Kubernetes API server.

### Service Account Creation

- **Inferred context (image `sa.png`):** likely shows the basic creation command and object, e.g. `kubectl create serviceaccount <name>` producing a `ServiceAccount` object, plus the automatically-created supporting Secret (on older Kubernetes versions) holding its token — establishing the basic "ServiceAccount is itself an API object, distinct from a regular User" concept.

### Service Accounts and Authentication

- When you want to access the Kubernetes API **from within a Pod**, you need to authenticate that Pod with the API server. **Service accounts** are used for this authentication process.
- When a service account is created in Kubernetes, a secret token is also created (on older versions). This token can be used as a **bearer token** to authenticate to the API server.
- By default, this token is **mounted as a volume in the pod** and is accessible at:
  ```
  /var/run/secrets/kubernetes.io/serviceaccount/token
  ```
  - **Inferred context (image `sa2.png`):** likely a Pod-internals diagram showing a volume named something like `<serviceaccount>-token-xxxxx` automatically mounted into every container in the Pod at `/var/run/secrets/kubernetes.io/serviceaccount/`, containing three files: `token` (the bearer JWT), `ca.crt` (the cluster CA cert, to verify the API server's TLS cert), and `namespace` (the Pod's namespace) — the classic "in-cluster client" bootstrap files that libraries like `client-go`'s in-cluster config auto-discover.
- By default, every Kubernetes namespace has a **`default`** service account. Every Pod in that namespace is automatically assigned this `default` service account unless otherwise specified.
- If you want to use a different service account for a specific Pod, specify it in the Pod specification:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
  spec:
    serviceAccountName: myapp-service-account
  ```
- To **opt out** of ServiceAccount token automounting:
  ```yaml
  spec:
    automountServiceAccountToken: false
  ```

### Enhanced Security Features (Kubernetes v1.22+)

- Service account tokens **do not have expiration or audience restrictions by default**. Kubernetes v1.22+ introduced improvements to enhance token security:
  - **Audience Bound** — the token is restricted to a specific audience (e.g., the Kubernetes API server).
  - **Time Bound** — the token has an expiration time (default of **1 hour**).
  - **Object Bound** — the token is bound to specific objects, such as a pod or service account.
  - Together these enhance security by limiting the scope of the token's validity and lifecycle.
- In v1.22, the **TokenRequest API** was introduced to request a token with a specific audience and expiration time. This API is used for obtaining tokens **instead of** using ServiceAccount token Secret objects. Tokens obtained via the TokenRequest API are more secure than those stored in Secret objects because they have a **bounded lifetime** and are **not readable by other API clients**.
- Also in this version, the service account token is injected as a **projected volume**, as shown below:
  - **Inferred context (image `sa3.png`):** likely a Pod-internals diagram similar to `sa2.png`, but showing a `projected` volume type combining multiple sources into one mount — the bound ServiceAccount token (with expiry/audience), the `ca.crt` ConfigMap, and a downward-API `namespace` file — reflecting the modern (post-1.22) `serviceAccountToken` projected-volume mechanism replacing the old auto-created long-lived Secret token.
- A Secret of type `kubernetes.io/service-account-token` is used to store a token credential that identifies a service account. **Since v1.22, this type of Secret is no longer used to mount credentials into Pods**, and obtaining tokens via the **TokenRequest API** is recommended instead. Tokens from the TokenRequest API are more secure than ones stored in Secret objects, because they have a bounded lifetime and are not readable by other API clients. You can use the **`kubectl create token`** command to obtain a token from the TokenRequest API.

### Creating Service Account Tokens

- In Kubernetes v1.24, **KEP-2799** introduced the **reduction of automatically created token secrets** for service accounts. You must now **manually** create service account token secrets using the `kubectl create token` command.
  ```bash
   kubectl create token <serviceaccount>
  ```
  - **Inferred context (image `sa4.png`):** likely shows sample output/usage of `kubectl create token <serviceaccount>` — a long JWT string printed to stdout, plus possibly flags like `--duration=<time>` to control the token's expiry, reinforcing that this is now the primary supported way to obtain a usable bearer token for a ServiceAccount on v1.24+.
- You should only create a service account token **Secret** object if you can't use the TokenRequest API to obtain a token, and the security exposure of persisting a non-expiring token credential in a readable API object is acceptable to you. For that:
  - Create a service account.
  - Create a secret for the service account.
  - In the secret, add an annotation `kubernetes.io/service-account.name: <serviceaccountname>`.
  - **Inferred context (image `sa5.png`):** likely shows a manual Secret manifest of `type: kubernetes.io/service-account-token` with the `metadata.annotations` field `kubernetes.io/service-account.name: <serviceaccountname>` set, and the resulting Secret's `data.token` field getting auto-populated by the control plane once created — the "old-style," now-manual-only, long-lived-token workflow.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/

---

## 22. Practice Test — Service Accounts

Solutions to the practice test — Service Accounts. Step-by-step lab-guide walkthrough:

1. **"How many service accounts exist in the default namespace?"**
   - Run:
     ```
     kubectl get serviceaccounts
     ```
   - Count the number of accounts returned.

2. **"What is the secret token used by the default service account?"**
   - Run:
     ```
     kubectl describe serviceaccount default
     ```
   - Look at the `Tokens` field.
   - **Answer:** `none`
   - (Reflects the v1.24+ KEP-2799 behavior from file 21 — the `default` service account no longer has an automatically-generated long-lived token Secret.)

3. **"We just deployed the Dashboard application. Inspect the deployment. What is the image used by the deployment?"**
   - Run:
     ```
     kubectl describe deployment
     ```
   - Look at the `Image` field.
   - **Answer:** `gcr.io/kodekloud/customimage/my-kubernetes-dashboard`

4. *(Information only step.)*

5. **"What is the state of the dashboard? Have the pod details loaded successfully?"**
   - Open the `web-dashboard` link located above the terminal and inspect the status. An error message is displayed.
   - **Answer:** `Failed`

6. **"What type of account does the Dashboard application use to query the Kubernetes API?"**
   - As evident from the error in the web-dashboard UI, the Pod makes use of a service account to query the Kubernetes API.
   - **Answer:** `Service Account`

7. **"Which account does the Dashboard application use to query the Kubernetes API?"**
   - To find this, inspect the YAML of the running Pod. The correct field for specifying a pod's service account is `serviceAccountName`. Use `grep` to extract only that field:
     ```
     kubectl get po -o yaml | grep 'serviceAccountName:'
     ```
   - Alternatively, use JSONPath. First get the Pod's name (it will be different each run):
     ```
     kubectl get pods
     ```
     Then:
     ```
     kubectl get po web-dashboard-65b9cf6cbb-79vbs -o jsonpath='{.spec.serviceAccountName}'
     ```
   - **Answer:** `default`

8. **"Inspect the Dashboard Application POD and identify the Service Account mounted on it."**
   - This is the same as the previous question.
   - **Answer:** `default`

9. **"At what location is the ServiceAccount credentials available within the pod?"**
   - Know that service account tokens are mounted in Pods as a volume mount — so look in the `volumeMounts` section.
     ```
     kubectl describe pod
     ```
   - Find the `Mounts` section (representing mounted volumes) — it shows a path to the mounted service account. From the answers, choose the one with the correct path prefix.
   - **Answer:** `/var/run/secrets`

10. **"Create a new ServiceAccount named `dashboard-sa`."**
    - Run:
      ```
      kubectl create serviceaccount dashboard-sa
      ```

11. *(Information only step.)*

12. **"Now we are going to test the service account's access to the dashboard."**
    1. Generate a token:
       ```
       kubectl create token dashboard-sa
       ```
       - This will generate a long string of characters.
    2. Select all the output using the mouse and copy it.
    3. Return to the dashboard UI, and paste this into the `Token` field.
    4. Press **Load Dashboard**. It should now display the pod.

13. **"Edit the deployment to change ServiceAccount from `default` to `dashboard-sa`."**
    1. Use `kubectl edit deployment web-dashboard`, which opens the running deployment in `vi`.
    2. Move down to the deployment spec and insert the service account as shown:
       ```yaml
       apiVersion: apps/v1
       kind: Deployment
       metadata:
         annotations:
           deployment.kubernetes.io/revision: "2"
         creationTimestamp: "2023-02-21T19:29:21Z"
         generation: 2
         name: web-dashboard
         namespace: default
         resourceVersion: "1499"
         uid: ac5a26bf-7a88-41cc-8db3-d5a4bd2ad31c
       spec:
         progressDeadlineSeconds: 600
         replicas: 1
         revisionHistoryLimit: 10
         selector:
           matchLabels:
             name: web-dashboard
         strategy:
           rollingUpdate:
             maxSurge: 25%
             maxUnavailable: 25%
           type: RollingUpdate
         template:
           metadata:
             creationTimestamp: null
             labels:
               name: web-dashboard
           spec:
             serviceAccountName: dashboard-sa    # <- Insert this line
             containers:
             - env:
               - name: PYTHONUNBUFFERED
                 value: "1"
               image: gcr.io/kodekloud/customimage/my-kubernetes-dashboard
               imagePullPolicy: Always
               name: web-dashboard
               ports:
               - containerPort: 8080
                 protocol: TCP
       ```
    3. Save and exit `vi`. The deployment will be updated.

14. **"Reload the dashboard and verify it works without pasting a token."**
    - Since the Pod's ServiceAccount is now `dashboard-sa` (which presumably has been granted the needed RBAC permissions, matching the successful token test in step 12), the dashboard should now load pod details successfully without any manual token entry.

**Exam-relevant takeaway:** `kubectl describe serviceaccount <name>` and `kubectl describe pod <name>` are the two commands to trace "which ServiceAccount is a Pod using" and "where is its token mounted." Since v1.24, `kubectl create token <serviceaccount>` is the standard way to mint a usable bearer token on demand (no more automatic long-lived Secret). Editing `spec.template.spec.serviceAccountName` in a Deployment (via `kubectl edit deployment`) is the mechanism to switch which identity a workload's Pods authenticate as.

---

*Continued in [kubernetes-security-notes_part_04.md](kubernetes-security-notes_part_04.md)
— Sections 23–30 (Image Security + Practice Test, Security Context + Practice Test,
Network Policies + Practice Test, kubectx/kubens, Download Presentation Deck) plus the
Quick Revision Checklist.*
