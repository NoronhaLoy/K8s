# Application Lifecycle Management — Complete Notes (Part 2 of 2)

> **Part 2 of 2** — covers Sections 10–18 (Secrets + Practice Test, Multi-Container Pods
> + Practice Test, Multi-Container Pod Design Patterns, Init Containers + Practice Test,
> Self-Healing Applications, Presentation Deck) plus the Quick Revision Checklist.
> Previous: [kubernetes-application-lifecycle-management-notes_part_01.md](kubernetes-application-lifecycle-management-notes_part_01.md)
> (Sections 01–09).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/05-Application-Lifecycle-Management/`

---

## 10. Secrets

### Web-Mysql Application

- **Inferred context (image `web.PNG`):** likely shows a simple two-tier architecture — a
  web application Pod connecting to a MySQL database Pod/Service, with the web app reading
  `DB_Host`, `DB_User`, `DB_Password` from its environment to make that connection.
- One way is to move the app's properties/envs into a ConfigMap — **but** a ConfigMap
  stores data in plain text. It is **definitely not** the right place to store a password:
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
   name: app-config
  data:
    DB_Host: mysql
    DB_User: root
    DB_Password: paswrd
  ```
  - **Inferred context (image `web1.PNG`):** likely highlights the `DB_Password: paswrd`
    line specifically, visually flagging it as the problem — plain text credentials
    sitting in a ConfigMap that anyone with `kubectl get configmap -o yaml` access can
    read directly.
- **Secrets** are used to store sensitive information. They are similar to ConfigMaps but
  are stored in an **encrypted format or a hashed format** *(course's phrasing — see the
  "Additional Notes" caveat below: in practice Secrets are only **base64-encoded**, not
  encrypted, unless you separately enable encryption at rest)*.

#### There are 2 steps involved with secrets

- **First**, create a secret.
- **Second**, inject the secret into a Pod.
- **Inferred context (image `sec.PNG`):** likely a simple two-box flow diagram
  (`Create Secret` → `Inject into Pod`), mirroring the ConfigMap two-phase workflow shown
  earlier.

#### There are 2 ways of creating a secret

**The Imperative way**
```
$ kubectl create secret generic app-secret --from-literal=DB_Host=mysql --from-literal=DB_User=root --from-literal=DB_Password=paswrd
$ kubectl create secret generic app-secret --from-file=app_secret.properties
```
- **Inferred context (image `csi.PNG`):** likely contrasts `--from-literal` (inline values)
  vs `--from-file` (read from a file), same pairing as the ConfigMap imperative options.

**The Declarative way**
- First, generate a base64-encoded hash value for each value, since Secret manifests store
  data pre-encoded:
  ```
  $ echo -n "mysql" | base64
  $ echo -n "root" | base64
  $ echo -n "paswrd"| base64
  ```
- Then create a secret definition file and deploy it with `kubectl create`:
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
   name: app-secret
  data:
    DB_Host: bX1zcWw=
    DB_User: cm9vdA==
    DB_Password: cGFzd3Jk
  ```
  ```
  $ kubectl create -f secret-data.yaml
  ```
- **Inferred context (image `csd.PNG`):** likely visually links each `echo -n ... |
  base64` output directly to the matching field in the YAML (`mysql` → `bX1zcWw=`, etc.),
  reinforcing that Secret manifests require values already encoded before you write them.

### Encode Secrets

- **Inferred context (image `enc.PNG`):** likely a diagram showing the encode step in
  isolation: `plaintext value` → `echo -n <value> | base64` → `base64 string` → pasted
  into the Secret's `data:` field. Emphasizes `-n` (no trailing newline) is required or the
  encoded value will be subtly wrong.

### View Secrets

- To view secrets (names only, values hidden):
  ```
  $ kubectl get secrets
  ```
- To describe a secret (still hides raw values, shows just key names and byte sizes):
  ```
  $ kubectl describe secret
  ```
- To view the actual (base64-encoded) values of the secret:
  ```
  $ kubectl get secret app-secret -o yaml
  ```
- **Inferred context (image `secv.PNG`):** likely shows sample output of `-o yaml`,
  displaying the `data:` block with base64 strings for each key — contrasted against the
  earlier `describe` output that only shows key names/sizes, not values.

### Decode Secrets

- To decode secret values back to plaintext:
  ```
  $ echo -n "bX1zcWw=" | base64 --decode
  $ echo -n "cm9vdA==" | base64 --decode
  $ echo -n "cGFzd3Jk" | base64 --decode
  ```
- **Inferred context (image `secd.PNG`):** likely the mirror-image diagram of the "Encode
  Secrets" one — `base64 string` → `base64 --decode` → back to original plaintext,
  emphasizing (as the course text does later) that **base64 is trivially reversible**, so
  this is *encoding*, not real encryption/security.

### Configuring secret with a pod

- To inject a secret into a Pod, add a new property **`envFrom`** followed by
  **`secretRef`** name, then create the Pod definition:
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
   name: app-secret
  data:
    DB_Host: bX1zcWw=
    DB_User: cm9vdA==
    DB_Password: cGFzd3Jk
  ```
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
      envFrom:
      - secretRef:
          name: app-secret
  ```
  ```
  $ kubectl create -f pod-definition.yaml
  ```
- **Inferred context (image `secp.PNG`):** likely shows the resulting container
  environment with `DB_Host`, `DB_User`, `DB_Password` populated (Kubernetes
  automatically base64-**decodes** values before injecting them as env vars — the app
  itself never has to decode anything).

#### There are other ways to inject secrets into pods

- You can inject as a **`Single ENV variable`** (one specific key, via
  `env[].valueFrom.secretKeyRef`).
- You can inject the whole secret as **files in a Volume**.
- **Inferred context (image `seco.PNG`):** likely a 3-way diagram identical in structure
  to the ConfigMap one (`cmp1.PNG`) but for Secrets: `envFrom.secretRef` (all keys),
  `env[].valueFrom.secretKeyRef` (one key), and a Secret-backed Volume mount (files).

### Secrets in pods as volume

- Each attribute in the secret is created as a **file**, with the value of the secret as
  its content.
- **Inferred context (image `secpv.PNG`):** likely shows a mounted directory (e.g.
  `/opt/app-secret-volume/`) containing one file per Secret key (`DB_Host`, `DB_User`,
  `DB_Password`), where `cat`-ing any of those files prints the **already-decoded**
  plaintext value — this is the classic pattern apps use to read credentials from disk
  instead of environment variables (env vars can leak via `/proc`, logs, `docker inspect`,
  etc., which volumes mitigate somewhat).

### Additional Notes: A Note on Secrets

- Secrets encode data in **base64** format. **Anyone with the base64-encoded secret can
  easily decode it.** As such, secrets can be considered **not very safe** by themselves.
- The concept of "safety" of Secrets is a bit confusing in Kubernetes. The
  [kubernetes documentation](https://kubernetes.io/docs/concepts/configuration/secret)
  page and many blogs refer to Secrets as a "safer option" for storing sensitive data —
  but it is not the Secret object *itself* that is safe, it is the **practices** around it.
- Secrets are **not encrypted**, so they are not inherently safer in that strict sense.
  However, best practices around using them make them safer, such as:
  - Not checking in secret object definition files to source code repositories.
  - [Enabling Encryption at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
    for Secrets so they are stored encrypted in etcd.
- Also, the way Kubernetes handles secrets provides some protection:
  - A secret is only sent to a node if a Pod on that node requires it.
  - Kubelet stores the secret in a **tmpfs** filesystem, so the secret is not written to
    disk storage.
  - Once the Pod that depends on the secret is deleted, kubelet deletes its local copy of
    the secret data as well.
- Read about [protections](https://kubernetes.io/docs/concepts/configuration/secret/#protections)
  and [risks](https://kubernetes.io/docs/concepts/configuration/secret/#risks) of using
  secrets in the official docs.
- There are better ways of handling sensitive data like passwords in Kubernetes, e.g.
  Helm Secrets, [HashiCorp Vault](https://www.vaultproject.io/).

### K8s Reference Docs

- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/concepts/configuration/secret/#use-cases
- https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/

---

## 11. Practice Test — Secrets

Step-by-step lab-guide walkthrough:

1. **"Run `kubectl get secrets` and count the number of [secrets]."**
   ```bash
   kubectl get secrets
   ```

2. **"Run `kubectl get secrets` and look at the DATA field."**
   ```bash
   kubectl get secrets
   ```
   - The `DATA` column shows the **count** of keys inside each secret (not their values).

3. **"Run `kubectl describe secret`."** (asked twice, likely for different secrets)
   ```bash
   kubectl describe secret <secret-name>
   ```
   - Shows key names and byte sizes only — values remain hidden even in `describe`.

4. **"We have already deployed the required pods and services. Check out the pods and
   services created. Check out the web application using the 'Webapp MySQL' link."**
   ```bash
   kubectl get pods
   kubectl get services
   ```
   - Visit the linked web app to observe its current state (likely failing to connect to
     the DB, since no secret is wired in yet).

5. **"Run command `kubectl create secret generic db-secret --from-literal=DB_Host=sql01
   --from-literal=DBUser=root --from-literal=DB_Password=password123`."**
   - **Corrected command** (note the key name typo `DBUser` in the prompt vs. the correct
     `DB_User` actually used in the solution):
   ```bash
   kubectl create secret generic db-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123
   ```

6. **"Check Answer at `/var/answers/answer-webapp.yaml`."**
   ```bash
   kubectl get pod webapp-pod -o yaml > web.yaml
   kubectl delete pod webapp-pod
   ```
   - Update `web.yaml` to add the secret injection under the container spec:
     ```yaml
     envFrom:
     - secretRef:
         name: db-secret
     ```
   ```bash
   kubectl create -f web.yaml
   ```

7. **"View the web application to verify it can successfully connect to the database."**
   - Reload the Webapp MySQL link — it should now successfully connect using the
     credentials injected from `db-secret`.

**Exam-relevant takeaway:** identical create-secret → get/delete/edit/create-pod →
`envFrom.secretRef` pattern as ConfigMaps, just swapping `configMapRef` for `secretRef` and
`kubectl create configmap` for `kubectl create secret generic`.

---

## 12. Multi-Container Pods

### Monolith and Microservices

- **Inferred context (image `loga.PNG`):** likely contrasts two architecture styles —
  **Monolithic** (a single large application process handling everything: web serving,
  business logic, logging, etc. in one codebase/container) vs. **Microservices**
  (functionality decomposed into small, independently deployable services, e.g. a web app
  service plus a separate logging/agent service) — this is the motivating context for why
  Kubernetes supports multiple containers cooperating tightly within one Pod.

#### Multi-Container Pods

- **Inferred context (image `mcp.PNG`):** likely shows the defining trait of a
  multi-container Pod: all containers in the same Pod share the same **network namespace**
  (same IP, can reach each other via `localhost`) and can share **storage volumes**,
  making them ideal for tightly-coupled helper processes (e.g. a log-shipping sidecar
  reading files written by the main app container via a shared volume).
- To create a new multi-container pod, add the new container's information to the Pod
  definition file:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: simple-webapp
    labels:
      name: simple-webapp
  spec:
    containers:
    - name: simple-webapp
      image: simple-webapp
      ports:
      - ContainerPort: 8080
    - name: log-agent
      image: log-agent
  ```
- **Inferred context (image `mcpc.PNG`):** likely shows the resulting
  `kubectl describe pod` / `kubectl get pod` output reporting **`READY: 2/2`**, listing
  both `simple-webapp` and `log-agent` as separate container entries under the same single
  Pod, each with its own `State`/`Ready` status, but sharing one Pod IP.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/

---

## 13. Practice Test — Multi-Container Pods

Step-by-step lab-guide walkthrough:

1. **"Identify the number of containers running in the 'red' pod."**
   ```bash
   kubectl get pod red
   ```
   - Read the `READY` column (e.g. `2/2` means 2 containers, both ready).

2. **"Identify the name of the containers running in the 'blue' pod."**
   ```bash
   kubectl describe pod blue
   ```
   - Read each `Containers:` sub-block's name.

3. **"[Create a pod matching the given spec.]"**
   ```bash
   kubectl create -f /var/answers/answer-yellow.yaml
   ```
   - **Answer file:** `/var/answers/answer-yellow.yaml`

4. **"We have deployed an application logging stack in the `elastic-stack` namespace.
   Inspect it."**
   ```bash
   kubectl get pods -n elastic-stack
   ```

5. **"Inspect the Kibana UI. There shouldn't be any logs for now."**
   - Visit the linked Kibana UI — expected to show **no data yet**, because the app Pod
     in this namespace has no logging sidecar shipping data to Elasticsearch yet.

6. **"Run `kubectl describe pod -n elastic-stack`."**
   ```bash
   kubectl describe pod -n elastic-stack
   ```
   - Inspect the Pod's container list — likely shows only a single `app` container so far
     (no sidecar), explaining why Kibana is empty.

7. **"Run `kubectl -n elastic-stack exec -it app cat /log/app.log`."**
   ```bash
   kubectl -n elastic-stack exec -it app cat /log/app.log
   ```
   - Confirms the app container **is** writing logs to `/log/app.log` locally — the
     problem is nothing is shipping that file's contents to Elasticsearch, motivating the
     need for a log-shipping sidecar container.

8. **"[Add a logging sidecar container to the app pod.]"**
   - **Answer file:** `/var/answers/answer-app.yaml`
   - Pattern: add a second container (e.g. filebeat/log-agent) to the Pod spec that
     mounts the **same volume** as the `app` container (so it can read `/log/app.log`)
     and forwards its contents to Elasticsearch/Logstash.

9. **"Inspect the Kibana UI. You should now see logs appearing in the 'Discover'
   section."**
   - After the sidecar is deployed and a moment for indexing, logs should appear. You
     might need to create an **index pattern** in Kibana first to browse them (linked
     video: https://bit.ly/2EXYdHf).

**Exam-relevant takeaway:** this lab is a live demonstration of the **sidecar pattern**
(covered conceptually in the next file) — a helper container added to an existing Pod,
sharing a volume with the main container, to ship logs out without modifying the main
application at all.

---

## 14. Multi-Container Pods Design Patterns

- This file consists of a link to the KodeKloud "Design Patterns" page and a single
  diagram image (`dp.PNG`) — no inline text. The following expands the three canonical
  multi-container Pod design patterns, referenced by the accompanying K8s blog link
  ("The Distributed System Toolkit: Patterns for Composite Containers").
- **Inferred context (image `dp.PNG`) — the three patterns it almost certainly depicts:**

  - **Sidecar pattern**
    - A helper container running **alongside** the main application container in the same
      Pod, extending/enhancing its functionality without modifying the main container's
      code.
    - Shares the Pod's network (`localhost`) and/or a mounted volume with the main
      container.
    - Classic example: the `log-agent` container from Section 12/13 — the main app writes
      logs to a shared volume, and the sidecar container ships those logs to a central
      logging backend (Elasticsearch/Logstash/Fluentd).
    - Other common sidecar uses: file/data sync containers, sync-and-push-to-git helpers.

  - **Adapter pattern**
    - A helper container that **standardizes/transforms** the main container's output
      into a common format expected by the outside world.
    - Example: several microservices might each emit monitoring data in a different,
      app-specific format; an adapter container sitting alongside each one transforms that
      output into a uniform format (e.g. Prometheus-compatible metrics) before it leaves
      the Pod, so the central monitoring system only needs to understand one format.

  - **Ambassador pattern**
    - A helper/proxy container that handles/simplifies **network communication** between
      the main container and the outside world (e.g. sharded/clustered external services,
      or different environments like dev/test/prod databases).
    - The main container always talks to `localhost:<port>`, and the ambassador container
      proxies that connection out to the real, possibly complex or environment-specific
      destination — so the main app never needs logic to know which actual backend/shard/
      environment it's really talking to.

- **Common thread across all three:** the pattern works because containers in the same
  Pod always share the same network namespace and (optionally) storage volumes — this is
  precisely the multi-container Pod mechanism introduced in Section 12.

### K8s Reference Docs

- https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/

---

## 15. Init Containers

- In a multi-container Pod, each container is expected to run a process that stays alive
  as long as the Pod's lifecycle. For example, in the web-app + logging-agent multi-
  container Pod discussed earlier, both containers are expected to stay alive at all
  times — the log agent's process is expected to stay alive as long as the web application
  is running. If either one fails, **the Pod restarts**.
- But sometimes you want to run a process that runs **to completion** in a container. For
  example:
  - A process that pulls code/binaries from a repository, to be used later by the main
    web application — a task that should run **only once**, when the Pod is first created.
  - A process that waits for an external service or database to be up **before** the
    actual application starts.
  - That's where **`initContainers`** come in.
- An init container is configured in a Pod like all other containers, except it is
  specified inside an **`initContainers`** section:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    labels:
      app: myapp
  spec:
    containers:
    - name: myapp-container
      image: busybox:1.28
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    initContainers:
    - name: init-myservice
      image: busybox
      command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ;']
  ```
- When a Pod is first created, the init container **runs first**, and the process inside
  it **must run to completion** before the real container hosting the application starts.
- You can configure **multiple** init containers, exactly like multi-container Pods. In
  that case, each init container runs **one at a time, in sequential order**.
- If any init container **fails to complete**, Kubernetes **restarts the Pod repeatedly**
  until the Init Container succeeds.
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    labels:
      app: myapp
  spec:
    containers:
    - name: myapp-container
      image: busybox:1.28
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    initContainers:
    - name: init-myservice
      image: busybox:1.28
      command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
    - name: init-mydb
      image: busybox:1.28
      command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
  ```
  - `init-myservice` waits until DNS resolution of `myservice` succeeds, then
    `init-mydb` waits until DNS resolution of `mydb` succeeds — **only after both
    complete**, in that order, does `myapp-container` start.

### K8s Reference Docs

- https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/

---

## 16. Practice Test — Init Containers

Step-by-step lab-guide walkthrough:

1. **"Identify the pod that has an initContainer configured."**
   ```bash
   kubectl get pods
   kubectl describe pods
   ```
   - Scan each Pod's `describe` output for an `Init Containers:` section — only Pods
     with one present have init containers configured.

2. **"What is the image used by the initContainer on the blue pod?"**
   ```bash
   kubectl describe pods blue
   ```
   - Read the `Init Containers > <name> > Image:` field.

3. **"Run `kubectl describe pod blue` and check the state field of the initContainer."**
   ```bash
   kubectl describe pod blue
   ```
   - Read `Init Containers > <name> > State:` — will show e.g. `Running`, `Waiting`, or
     `Terminated`.

4. **"Check the reason field of the initContainer."**
   ```bash
   kubectl describe pod blue
   ```
   - Read the `Reason:` sub-field under the init container's state — e.g.
     `PodInitializing` while an init container is still executing, or a failure reason
     such as `Error`/`CrashLoopBackOff` if it's failing.

5. **"Run `kubectl describe pod purple`."**
   ```bash
   kubectl describe pod purple
   ```
   - General inspection of a Pod with **multiple** init containers configured.

6. **"Run the `kubectl describe pod purple` command and look at the container state."**
   ```bash
   kubectl describe pod purple
   ```
   - The main application container's state will show `Waiting` with reason
     `PodInitializing` for as long as any init container hasn't yet completed — this is
     the direct, observable proof that init containers block the main container's start.

7. **"Check the commands used in the initContainers. The first one sleeps for 600
   seconds (10 minutes) and the second one sleeps for 1200 seconds (20 minutes)."**
   ```bash
   kubectl describe pod purple
   ```
   - Read each init container's `Command:` field to confirm the sleep durations —
     illustrates the **sequential** execution model: the second init container (20 min
     sleep) doesn't even start until the first (10 min sleep) finishes, so the main
     container won't start for **30 minutes total**.

8. **"Update the pod red to use an initContainer that uses the busybox image and sleeps
   for 20 seconds."**
   ```bash
   kubectl get pod red -o yaml > red.yaml
   kubectl delete pod red
   ```
   - Edit `red.yaml` to add:
     ```yaml
     initContainers:
     - name: init-red
       image: busybox
       command: ['sh', '-c', 'sleep 20']
     ```
   ```bash
   kubectl create -f red.yaml
   ```

9. **"Check the command used by the initContainer. Looks like there is a typo in sleep
   command. Fix it — it should be `sleep 2` not `sleeeep 2`."**
   ```bash
   kubectl describe pod orange
   kubectl get pod orange -o yaml > orange.yaml
   kubectl delete pod orange
   ```
   - Fix the typo in `orange.yaml`'s init container command (`sleeeep` → `sleep`), then:
   ```bash
   kubectl create -f orange.yaml
   ```
   - **Why this matters:** an invalid/misspelled command inside an init container causes
     it to error out and Kubernetes to **restart the Pod repeatedly**, leaving the main
     application container permanently stuck `Waiting`/`PodInitializing` until the typo is
     fixed — a very common real-world debugging scenario for init containers.

**Exam-relevant takeaway:** `kubectl describe pod <name>` is again the one command that
answers everything about init containers — image, state, reason, and exact command — and
init containers, like `command`/`args`, require the delete-edit-recreate cycle to fix.

---

## 17. Self-Healing Applications

- Kubernetes supports **self-healing applications** through **ReplicaSets** and
  **Replication Controllers**.
- The replication controller helps ensure that a Pod is **re-created automatically** when
  the application within the Pod **crashes**. It helps ensure enough replicas of the
  application are running **at all times**.
- Kubernetes provides additional support to check the **health** of applications running
  within Pods, and take necessary actions, through **Liveness and Readiness Probes**.
  However, these are **not required for the CKA exam** and, as such, are **not covered
  here** — these are topics for the **Certified Kubernetes Application Developer (CKAD)**
  exam and are covered in the CKAD course.
- **Supplementary/general context (not explicit in this short lecture, but standard
  companion knowledge for "self-healing" and commonly tested alongside this topic):** Pod
  self-healing also depends on the Pod's `spec.restartPolicy`, which governs whether the
  **kubelet** restarts containers in that Pod on exit:
  - `Always` (default) — always restart the container on exit, regardless of exit code.
  - `OnFailure` — restart only if the container exits with a non-zero (failure) exit code.
  - `Never` — never restart the container automatically.
  - This is a Pod-level field distinct from the ReplicaSet/Deployment-level self-healing
    described above (which recreates a whole *missing* Pod); `restartPolicy` governs
    restarting a *container* within an existing Pod.

---

## 18. Download Presentation Deck

- The section provides a downloadable slide deck accompanying the video lectures:
  [Presentation Deck](https://kodekloud.com/topic/download-presentation-deck-4/).
- Purely a resource/reference link — no technical content of its own.

---

## Quick Revision Checklist

- [ ] **Rolling Updates & Rollback**
  - Deployment strategies: `Recreate` (all old Pods down, then all new Pods up — causes
    downtime) vs `RollingUpdate` (default; incremental replace — no downtime).
  - `kubectl rollout status deployment/<name>` / `kubectl rollout history
    deployment/<name>` / `kubectl rollout undo deployment/<name>`.
  - `kubectl apply -f <file>` (declarative update) vs `kubectl set image
    deployment/<name> <container>=<image>` (imperative shortcut) both trigger a new
    revision/rollout.
  - Each rollout creates a **new ReplicaSet**; old ReplicaSets are kept (scaled to 0) for
    rollback/history.
  - `RollingUpdateStrategy` has `maxUnavailable` / `maxSurge` — these fields only make
    sense under `strategy.type: RollingUpdate`; must be **removed** if you switch
    `strategy.type` to `Recreate`.

- [ ] **Command vs Args vs Entrypoint precedence**
  - Docker `ENTRYPOINT` = executable → maps to Pod `command:`.
  - Docker `CMD` = default parameters → maps to Pod `args:`.
  - Pod `command`/`args`, when set, **fully override** (not merge with) the image's
    `ENTRYPOINT`/`CMD`.
  - `command`/`args` array elements must be **strings** (quote numbers, e.g. `"5000"`).
  - Appending extra words to `docker run <image> <words>` overrides just the image's
    `CMD`, not its `ENTRYPOINT`; use `docker run --entrypoint` to override the entrypoint
    itself.
  - `command`/`args` are **immutable** on a running Pod — get YAML → delete Pod → edit →
    recreate.

- [ ] **Environment Variables**
  - Plain literal: `env: [{name: X, value: Y}]`.
  - From ConfigMap: `envFrom: [{configMapRef: {name: ...}}]` (all keys) or
    `env[].valueFrom.configMapKeyRef` (one key).
  - From Secret: `envFrom: [{secretRef: {name: ...}}]` (all keys) or
    `env[].valueFrom.secretKeyRef` (one key).

- [ ] **ConfigMaps**
  - Imperative: `kubectl create configmap <name> --from-literal=K=V` or
    `--from-file=<file>`.
  - Declarative: `ConfigMap` manifest with `data:` map + `kubectl create -f`.
  - Injection options: `envFrom.configMapRef` (all), `env[].valueFrom.configMapKeyRef`
    (one), or **Volume** (each key becomes a file).
  - `kubectl get configmaps`/`kubectl get cm`, `kubectl describe configmaps`.

- [ ] **Secrets & Encoding**
  - Imperative: `kubectl create secret generic <name> --from-literal=K=V` or
    `--from-file=<file>`.
  - Declarative: values must be pre-**base64-encoded** (`echo -n "value" | base64`,
    the `-n` matters) in the `data:` field.
  - Decode: `echo -n "<b64>" | base64 --decode`.
  - Injection identical shape to ConfigMaps: `envFrom.secretRef`,
    `env[].valueFrom.secretKeyRef`, or Volume (each key → decoded-content file).
  - **Secrets are base64-encoded, NOT encrypted** — trivially reversible; true security
    requires Encryption at Rest for etcd + access-control best practices, not the Secret
    object alone.
  - Kubelet stores Secret data in **tmpfs** (not written to disk) and deletes it when the
    dependent Pod is deleted; a Secret is only ever sent to nodes that actually need it.

- [ ] **Multi-Container Pod Design Patterns**
  - Containers in the same Pod share the same **network namespace** (localhost) and can
    share **Volumes**.
  - **Sidecar** — helper container extends the main container (e.g. log shipper reading a
    shared volume).
  - **Adapter** — helper container standardizes/transforms the main container's output
    into a common format.
  - **Ambassador** — helper/proxy container simplifies the main container's outbound
    network calls to potentially complex/sharded/environment-specific external services.
  - Multi-container Pod `READY` column reads `N/N` for N containers; use
    `kubectl logs <pod> <container>` and `kubectl exec <pod> -c <container> ...` once
    there's more than one container.

- [ ] **Init Containers**
  - Declared under `initContainers:`, separate from `containers:`.
  - Must **run to completion** before any regular container in the Pod starts.
  - Multiple init containers run **sequentially, one at a time**, in the order listed.
  - If an init container fails, Kubernetes **restarts the whole Pod repeatedly** until it
    succeeds — main container stays `Waiting` with reason `PodInitializing` the whole time.
  - `kubectl describe pod <name>` shows `Init Containers:` section with per-container
    `Image`, `State`, `Reason`, and `Command` — the primary diagnostic command for this
    topic.

- [ ] **Restart Policies / Self-Healing**
  - **ReplicaSets/Replication Controllers** provide Pod-level self-healing — automatically
    recreate a Pod if it's deleted/crashes, and maintain the desired replica count.
  - **Liveness/Readiness Probes** exist for finer-grained health checking but are
    explicitly **out of scope for the CKA exam** (CKAD topic).
  - `spec.restartPolicy` (`Always` default / `OnFailure` / `Never`) governs container-level
    restart behavior within a Pod (general Kubernetes knowledge that complements this
    topic, though not spelled out in the source lecture itself).

---

*End of Application Lifecycle Management notes.*
