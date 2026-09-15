# Application Lifecycle Management — Complete Notes (Part 1 of 2)

> **Part 1 of 2** — covers Sections 01–09 (Section Introduction, Rolling Updates &
> Rollback + Practice Test, Commands & Arguments in Docker/Kubernetes + Practice Test,
> Environment Variables, ConfigMaps + Practice Test).
> Next: [kubernetes-application-lifecycle-management-notes_part_02.md](kubernetes-application-lifecycle-management-notes_part_02.md)
> (Sections 10–18 + Quick Revision Checklist).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/05-Application-Lifecycle-Management/`

---

## 01. Application Lifecycle Management — Section Introduction

- Video reference: *Application Lifecycle Management Section Introduction* (KodeKloud).
- This section of the CKA course covers four themes, all built on top of Deployments:
  - **Rolling Updates and Rollbacks** in Deployments
  - **Configure Applications** (commands/args, env vars, ConfigMaps, Secrets)
  - **Scale Applications**
  - **Self-Healing Application** (ReplicaSets/Replication Controllers)
- Big picture: this section is about what happens to an application *after* it's initially
  deployed — how you update it safely, configure it, and keep it running automatically.

---

## 02. Rolling Updates and Rollback

### Rollout and Versioning in a Deployment

- Every time the contents of a Deployment (e.g. the container image) are updated, a new
  **rollout** is triggered, and this rollout creates a new **Deployment Revision** —
  revisions are named `revision 1`, `revision 2`, etc.
- **Inferred context (image `rollv.PNG`):** the diagram most likely shows that a Deployment
  does not manage Pods directly — it manages a **ReplicaSet**. On each rollout, Kubernetes
  creates a **brand-new ReplicaSet** (rather than mutating the existing Pods in place) and
  scales it up while scaling the old ReplicaSet down to zero. Kubernetes keeps the old
  ReplicaSet objects around (scaled to 0) so that revision history is available for
  rollback — this is exactly what `kubectl rollout history` reads back later.

### Rollout commands

- See the live status of an in-progress rollout:
  ```
  $ kubectl rollout status deployment/myapp-deployment
  ```
- See revision history:
  ```
  $ kubectl rollout history deployment/myapp-deployment
  ```
- **Inferred context (image `rollc.PNG`):** likely shows sample output for both commands —
  `rollout status` printing progressive lines such as *"Waiting for deployment
  ... rollout to finish: 2 of 3 new replicas have been updated..."* ending in
  *"deployment ... successfully rolled out"*; and `rollout history` printing a table with
  columns `REVISION` and `CHANGE-CAUSE` listing each past update.

### Deployment Strategies

- There are **2 types of deployment strategies**:
  1. **Recreate**
  2. **RollingUpdate** (the **Default Strategy**)
- **Inferred context (image `dst.PNG`):** the diagram most likely contrasts the two
  strategies visually:
  - **Recreate** — all existing (old-version) Pods are **destroyed first**, then all new
    Pods are created. This causes application **downtime** in the gap between destroying
    the old set and the new set becoming available.
  - **RollingUpdate** — old Pods are taken down and new Pods are brought up **one/few at a
    time** (a mix of old and new Pods coexist temporarily). The application stays
    accessible throughout the update — **no downtime**, which is why it's the default.

### `kubectl apply` — updating a Deployment declaratively

- To update a Deployment: edit the deployment definition file with the necessary changes,
  save it, then run:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
   name: myapp-deployment
   labels:
    app: nginx
  spec:
   template:
     metadata:
       name: myap-pod
       labels:
         app: myapp
         type: front-end
     spec:
      containers:
      - name: nginx-container
        image: nginx:1.7.1
   replicas: 3
   selector:
    matchLabels:
      type: front-end
  ```
  ```
  $ kubectl apply -f deployment-definition.yaml
  ```
- Alternate way to update a deployment (e.g. bump just the image tag) without editing the
  YAML file:
  ```
  $ kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
  ```
  - **Inferred context (image `ka.PNG`):** likely illustrates that `kubectl set image` is
    a quick imperative shortcut that produces the *same effect* as editing the YAML's
    `image:` field and re-applying — it triggers a new rollout/revision immediately,
    without needing to touch a manifest file on disk.

### Recreate vs RollingUpdate

- **Inferred context (image `rcrl.PNG`):** likely a side-by-side timeline:
  - **Recreate timeline:** `[v1 pod][v1 pod][v1 pod]` → all terminated → **gap (app down)**
    → `[v2 pod][v2 pod][v2 pod]` all created together.
  - **RollingUpdate timeline:** `[v1][v1][v1]` → `[v2][v1][v1]` → `[v2][v2][v1]` →
    `[v2][v2][v2]`, with at least one Pod always serving traffic — no gap, no downtime.

### Upgrades

- **Inferred context (image `up.PNG`):** likely shows what physically happens on an
  upgrade at the ReplicaSet level: the **old ReplicaSet is scaled down** to `0` replicas
  while a **new ReplicaSet is scaled up** to the desired replica count, one/few Pods at a
  time (for RollingUpdate) — this is the mechanism underlying both `kubectl apply` and
  `kubectl set image`.

### Rollback

- **Inferred context (image `rb.PNG`):** likely shows the reverse of the "Upgrades"
  diagram — on rollback, the **new ReplicaSet is scaled back down to 0** and the
  **previous ReplicaSet is scaled back up**, restoring the Pods that existed under the
  prior revision.
- To undo a change (roll back to the previous revision):
  ```
  $ kubectl rollout undo deployment/myapp-deployment
  ```

### `kubectl create`

- To create a deployment imperatively:
  ```
  $ kubectl create deployment nginx --image=nginx
  ```

### Summarize kubectl commands

```
$ kubectl create -f deployment-definition.yaml
$ kubectl get deployments
$ kubectl apply -f deployment-definition.yaml
$ kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
$ kubectl rollout status deployment/myapp-deployment
$ kubectl rollout history deployment/myapp-deployment
$ kubectl rollout undo deployment/myapp-deployment
```

- **Inferred context (image `sum.PNG`):** likely a one-slide visual cheat-sheet
  reproducing the exact command list above as a quick-reference summary graphic.

### K8s Reference Docs

- https://kubernetes.io/docs/concepts/workloads/controllers/deployment
- https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment

---

## 03. Practice Test — Rolling Updates and Rollback

Step-by-step lab-guide walkthrough:

1. **"We have deployed a simple web application. Inspect the PODs and the Services."**
   ```bash
   kubectl get pods
   kubectl get services
   ```
   - Confirms the app and its exposing Service exist and are `Running`/have an endpoint.

2. **"What is the current color of the web application?"**
   - Access the web application through its exposed Service/portal in the browser (or via
     the provided UI tab) and visually read the color shown on the page.

3. **"Execute the script at `/root/curl-test.sh`."**
   ```bash
   /root/curl-test.sh
   ```
   - This script typically curls the Service endpoint repeatedly to sample which
     color/version is currently answering requests — useful later to *prove* whether a
     rolling update caused a mix of old/new responses or a clean cut-over.

4. **"Run `kubectl describe deployment` and look at 'Desired Replicas'."**
   ```bash
   kubectl describe deployment
   ```
   - Read the `Replicas:` line (format similar to `Replicas: 3 desired | 3 updated | 3
     total | 3 available | 0 unavailable`) — the **desired** count is the answer.

5. **"Run `kubectl describe deployment` and look for 'Images'."**
   ```bash
   kubectl describe deployment
   ```
   - Read the `Pod Template > Containers > Image:` field to identify the exact image/tag
     currently in use.

6. **"Run `kubectl describe deployment` and look at 'StrategyType'."**
   ```bash
   kubectl describe deployment
   ```
   - Read the `StrategyType:` field — will read `RollingUpdate` (default) or `Recreate`.

7. **"If you were to upgrade the application now what would happen?"**
   - **Answer:** *PODs are upgraded few at a time* — because the deployment's strategy is
     `RollingUpdate`, Kubernetes replaces old Pods with new ones incrementally rather than
     all at once, keeping the app available during the upgrade.

8. **"Run `kubectl edit deployment frontend` and modify the required field."**
   ```bash
   kubectl edit deployment frontend
   ```
   - Opens the live Deployment object in an editor (typically to change the container
     `image:` tag) — saving triggers an immediate new rollout.

9. **"Execute the script at `/root/curl-test.sh`."**
   ```bash
   /root/curl-test.sh
   ```
   - Re-run the same probe script to observe the update in progress/completed (e.g. now
     returning the new color).

10. **"Look at the Max Unavailable value under RollingUpdateStrategy in deployment
    details."**
    ```bash
    kubectl describe deployment
    ```
    - Read the `RollingUpdateStrategy:` line, e.g. `25% max unavailable, 25% max surge` —
      report the `maxUnavailable` value shown.

11. **"Run `kubectl edit deployment frontend` and modify the required field. Make sure to
    delete the properties of rollingUpdate as well, set at `strategy.rollingUpdate`."**
    ```bash
    kubectl edit deployment frontend
    ```
    - This is the **Recreate** conversion step: change `strategy.type` to `Recreate`, and
      because `rollingUpdate.maxUnavailable`/`rollingUpdate.maxSurge` are only valid under
      the `RollingUpdate` strategy, they must be **deleted** — leaving them in place while
      `strategy.type: Recreate` is set is invalid/ignored and is a common exam trap.

12. **"Run `kubectl edit deployment frontend` and modify the required field."**
    ```bash
    kubectl edit deployment frontend
    ```
    - A further edit (e.g. bump the image again) — this time under the `Recreate`
      strategy, so all old Pods terminate before any new Pods are created.

13. **"Execute the script at `/root/curl-test.sh`."**
    ```bash
    /root/curl-test.sh
    ```
    - Running the probe again during this update should show a **gap/downtime window**
      where the app is briefly unreachable — proving the practical difference between
      `Recreate` and `RollingUpdate` observed earlier.

**Exam-relevant takeaway:** `kubectl describe deployment` is the single command that
answers almost every question in this lab — desired replicas, image, strategy type, and
`maxUnavailable`/`maxSurge` all live in its output. Editing `strategy.type` between
`Recreate` and `RollingUpdate` requires also adding/removing the `rollingUpdate:` block
consistently.

---

## 04. Commands and Arguments in Docker

- To run a docker container:
  ```
  $ docker run ubuntu
  ```
- To list running containers:
  ```
  $ docker ps
  ```
- To list **all** containers, including stopped ones:
  ```
  $ docker ps -a
  ```
- **Inferred context (image `dc.PNG`):** likely shows sample `docker ps` vs `docker ps -a`
  output side by side — `docker ps` shows only containers with `STATUS: Up ...`, while
  `docker ps -a` additionally lists containers with `STATUS: Exited (0) ...`, illustrating
  that a plain `docker run ubuntu` container **exits immediately** because Ubuntu's default
  image has no long-running foreground process — this sets up the whole topic.

#### Unlike virtual machines, containers are not meant to host an operating system

- Containers are meant to run a **specific task or process**, such as hosting an instance
  of a webserver, application server, or database server, etc.
- **Inferred context (image `ex.PNG`):** likely shows a short list of example images and
  the single process each one runs as its main/PID-1 process (e.g. `nginx` → runs the
  nginx web server process; `mysql` → runs the mysqld process) — reinforcing that a
  container's lifecycle is tied to that one foreground process: when it exits, the
  container exits.

#### How do you specify a different command to start the container?

- One option: append a command to the `docker run` command — this **overrides** the
  default command specified within the image:
  ```
  $ docker run ubuntu sleep 5
  ```
- This way, when the container starts it runs the `sleep` program, waits 5 seconds, and
  then exits. How do you make that change **permanent**?
  - **Inferred context (image `sleep.PNG`):** likely shows the Dockerfile `CMD`
    instruction as the permanent equivalent of appending `sleep 5` at the command line —
    e.g. `CMD sleep 5` baked into a custom image, so every `docker run <image>` (with no
    extra args) behaves like the ad-hoc override did.
- There are different ways of specifying the command: either as plain shell form, or in
  JSON array format.
  - **Inferred context (image `sleep1.PNG`):** likely contrasts:
    - **Shell form:** `CMD sleep 5`
    - **Exec/JSON array form:** `CMD ["sleep", "5"]`
    - The array form is required when you also want to combine it with `ENTRYPOINT` (each
      element becomes one argument, avoiding shell-parsing ambiguity).
- Build the docker image:
  ```
  $ docker build -t ubuntu-sleeper .
  ```
- Run the docker container:
  ```
  $ docker run ubuntu-sleeper
  ```
  - **Inferred context (image `sleep2.PNG`):** likely demonstrates the finished
    `ubuntu-sleeper` image using `ENTRYPOINT ["sleep"]` with `CMD ["5"]` as the default
    parameter — so `docker run ubuntu-sleeper` sleeps 5 seconds by default, while
    `docker run ubuntu-sleeper 10` overrides just the `CMD` portion (`10` replaces `5`)
    and sleeps 10 seconds instead, without needing to touch the `ENTRYPOINT`.

### Entrypoint Instruction

- The **`ENTRYPOINT`** instruction is like the `CMD` instruction, in that you specify the
  program that will run when the container starts — **and** whatever you specify on the
  command line when running the container gets **appended** to (not replacing) the
  `ENTRYPOINT` command as its parameters.
- (General Docker knowledge, consistent with the source): to override `ENTRYPOINT` itself
  at runtime you must use `docker run --entrypoint <new-command> <image>`.

### K8s Reference Docs

- https://docs.docker.com/engine/reference/builder/#cmd

---

## 05. Commands and Arguments in Kubernetes

- Anything that is appended to the `docker run` command goes into the **`args`** property
  of the Pod definition file, in the form of an array.
- The Pod `command` field corresponds to the **`ENTRYPOINT`** instruction in the
  Dockerfile. So, to summarize, there are **2 fields** that correspond to **2
  instructions** in the Dockerfile:

  | Dockerfile instruction | Pod spec field |
  |---|---|
  | `ENTRYPOINT` | `command` |
  | `CMD` | `args` |

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: ubuntu-sleeper-pod
  spec:
   containers:
   - name: ubuntu-sleeper
     image: ubuntu-sleeper
     command: ["sleep2.0"]
     args: ["10"]
  ```
- **Inferred context (image `args.PNG`):** likely a mapping/precedence table reinforcing:
  - Pod `command:` **overrides** the image's `ENTRYPOINT` entirely (not merged/appended).
  - Pod `args:` **overrides** the image's `CMD` entirely.
  - If `command` is set in the Pod but `args` is not, the image's default `CMD` is
    discarded (not preserved) — you must explicitly re-specify `args` if you still want
    parameters.
  - **Exam-relevant:** command/args values in a Pod spec **must be YAML strings** — bare
    numbers (e.g. `args: [10]`) are invalid; must be written as `args: ["10"]`.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/

---

## 06. Practice Test — Commands and Arguments

Step-by-step lab-guide walkthrough:

1. **"Run `kubectl get pods` and count the number of pods."**
   ```bash
   kubectl get pods
   ```
   - Count entries in the output list.

2. **"Run `kubectl describe pod` and look for command option."**
   ```bash
   kubectl describe pod <pod-name>
   ```
   - Read the `Command:` field under the container's section — shows the effective
     entrypoint/command currently configured (or blank if the image's default is used).

3. **"Set the command option to `['sleep', '5000']`."**
   - Since a running Pod's `command` field is **immutable**, the standard pattern is:
     ```bash
     kubectl get pod <pod-name> -o yaml > pod.yaml
     kubectl delete pod <pod-name>
     # edit pod.yaml: spec.containers[].command: ["sleep", "5000"]
     kubectl create -f pod.yaml
     ```
   - **Answer file:** `/var/answers/answer-ubuntu-sleeper-2.yaml`

4. **"Both `sleep` and `1200` should be defined as a string."**
   - Reinforces the YAML-typing trap: write `command: ["sleep", "1200"]` (both elements
     quoted as strings), **not** `command: [sleep, 1200]` with a bare unquoted numeral,
     which YAML would otherwise interpret as an integer and Kubernetes would reject/behave
     unexpectedly for a `command`/`args` array (which expects strings).
   - **Answer file:** `/var/answers/answer-ubuntu-sleeper-3.yaml`

5. **Further variant of the same task.**
   - **Answer file:** `/var/answers/answer-ubuntu-sleeper-3-2.yaml`

6. **"Inspect the file `Dockerfile` given at `/root/webapp-color`. What command is run at
   container startup?"**
   ```bash
   cat /root/webapp-color/Dockerfile
   ```
   - **Answer:** `python app.py` (the image's default `ENTRYPOINT`/`CMD`, with no
     color argument, i.e. the app's built-in default color).

7. **"Inspect the file `Dockerfile2` given at `/root/webapp-color`. What command is run at
   container startup?"**
   ```bash
   cat /root/webapp-color/Dockerfile2
   ```
   - **Answer:** `python app.py --color red` (this Dockerfile bakes in `--color red` as a
     default `CMD` argument to the `ENTRYPOINT`).

8. **"The `command` (entrypoint) is overridden in the pod definition."**
   - **Answer:** `--color green` — because the Pod spec's `command`/`args` fields take
     precedence over anything baked into the Dockerfile, the Pod definition overrides the
     Dockerfile2 default (`red`) and forces `green` instead.

9. **"Inspect the two files under directory `webapp-color-3`. What command is run at
   container startup?"**
   ```bash
   cat /root/webapp-color-3/Dockerfile
   cat /root/webapp-color-3/pod-definition.yaml
   ```
   - **Answer:** `python app.py --color pink` — the Dockerfile sets `ENTRYPOINT
     ["python", "app.py"]`, and the Pod's `args: ["--color", "pink"]` supplies the runtime
     parameter, resulting in `python app.py --color pink`.

10. **Final task — produce a Pod that starts the webapp with `--color green`.**
    - **Answer file:** `/var/answers/answer-webapp-color-green.yaml`
    - Pattern: set `command`/`args` in the Pod spec (or just `args` if the Dockerfile's
      `ENTRYPOINT` is already `python app.py`) to `["--color", "green"]`.

**Exam-relevant takeaway:** the whole lab is testing the same skill repeatedly — reading a
Dockerfile's `ENTRYPOINT`/`CMD` to predict the *default* startup command, then reading/
editing a Pod's `command`/`args` to know what actually happens once Kubernetes overrides
it. Since `command`/`args` are immutable on a live Pod, always **get YAML → delete →
edit → recreate**.

---

## 07. Configure Environment Variables in Applications

#### ENV variables in Docker

```
$ docker run -e APP_COLOR=pink simple-webapp-color
```

#### ENV variables in Kubernetes

- To set an environment variable, set an **`env`** property in the Pod definition file:
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
     env:
     - name: APP_COLOR
       value: pink
  ```
- **Inferred context (image `env.PNG`):** likely shows the `env:` list format itself —
  each entry is an object with `name:`/`value:` keys, and multiple env vars are simply
  multiple list items under `env:`.
- There are other ways of setting environment variables, such as **`ConfigMaps`** and
  **`Secrets`**.
  - **Inferred context (image `cms.PNG`):** likely a small diagram showing three ways to
    populate a container's environment: (1) a **plain literal value** directly in the Pod
    spec (as above), (2) a value sourced from a **ConfigMap** (for non-sensitive config),
    and (3) a value sourced from a **Secret** (for sensitive data) — foreshadowing the
    next two lectures.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/

---

## 08. Configure ConfigMaps in Applications

### ConfigMaps

- There are **2 phases** involved in configuring ConfigMaps:
  1. **First**, create the ConfigMap.
  2. **Second**, inject it into the Pod.
- There are **2 ways** of creating a ConfigMap:

  **The Imperative way**
  ```
  $ kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod
  $ kubectl create configmap app-config --from-file=app_config.properties (Another way)
  ```
  - **Inferred context (image `cmi.PNG`):** likely contrasts `--from-literal` (inline
    key=value pairs, good for a handful of simple values) against `--from-file` (reads an
    entire properties/config file and turns each line into a key/value pair — good for
    bulk config).

  **The Declarative way**
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
   name: app-config
  data:
   APP_COLOR: blue
   APP_MODE: prod
  ```
  ```
  Create a config map definition file and run the 'kubectl create` command to deploy it.
  $ kubectl create -f config-map.yaml
  ```
  - **Inferred context (image `cmd1.PNG`):** likely just visually reiterates this
    YAML-file → `kubectl create -f` workflow, paralleling how Pods/Deployments are
    created declaratively.

### View ConfigMaps

- To view ConfigMaps:
  ```
  $ kubectl get configmaps (or)
  $ kubectl get cm
  ```
- To describe a ConfigMap:
  ```
  $ kubectl describe configmaps
  ```
- **Inferred context (image `cmv.PNG`):** likely shows sample `describe` output listing
  the ConfigMap's `Data` section with each key and its plain-text value (e.g.
  `APP_COLOR: blue`), demonstrating that ConfigMap data is stored/viewed as **plain
  text**, unlike Secrets.

### ConfigMap in Pods

- Inject a ConfigMap into a Pod via **`envFrom`**:
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
     - configMapRef:
         name: app-config
  ```
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-config
  data:
    APP_COLOR: blue
    APP_MODE: prod
  ```
  ```
  $ kubectl create -f pod-definition.yaml
  ```
- **Inferred context (image `cmp.PNG`):** likely shows the result — every key in the
  ConfigMap's `data:` becomes an environment variable inside the container (here,
  `APP_COLOR=blue` and `APP_MODE=prod` both appear in the container's environment),
  confirming `envFrom` injects **all** keys at once.

#### There are other ways to inject configuration variables into a pod

- You can inject it as a **`Single Environment Variable`** (pick one specific key out of
  the ConfigMap rather than all of them, via `env[].valueFrom.configMapKeyRef`).
- You can inject it as a file in a **`Volume`** (mount the whole ConfigMap as a
  directory of files, one file per key).
- **Inferred context (image `cmp1.PNG`):** likely a 3-way diagram comparing these three
  injection styles side-by-side: (1) `envFrom.configMapRef` = inject every key as an env
  var, (2) `env[].valueFrom.configMapKeyRef` = inject one specific key as one named env
  var, (3) `volumes`/`volumeMounts` with a ConfigMap volume source = expose every key as
  a file inside a mounted directory.

### K8s Reference Docs

- https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-container-environment-variables-using-configmap-data
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files

---

## 09. Practice Test — Environment Variables

Step-by-step lab-guide walkthrough:

1. **"Run `kubectl get pods` and count the number of pods."**
   ```bash
   kubectl get pods
   ```

2. **"Run `kubectl describe pod` and look for ENV option."** (asked twice, for different
   Pods/values)
   ```bash
   kubectl describe pod <pod-name>
   ```
   - Read the `Environment:` section of the container's description to see currently
     configured env vars and their values.

3. **"View the web application UI by clicking on the 'Webapp Color' Tab above your
   terminal."**
   - Confirms visually which color the app currently renders (should match the env var
     inspected above).

4. **"Set the environment option to `APP_COLOR` = green."**
   - Since env vars are immutable on a running Pod, use the standard get-delete-edit-
     recreate pattern:
   ```bash
   kubectl get pods webapp-color -o yaml > green.yaml
   kubectl delete pods webapp-color
   # Update APP_COLOR to green inside green.yaml
   kubectl create -f green.yaml
   ```

5. **"View the changes to the web application UI."**
   - Reload the Webapp Color tab — should now render green.

6. **"Run `kubectl get configmaps`."**
   ```bash
   kubectl get configmaps
   ```

7. **"Run `kubectl describe configmaps` and look for `DB_HOST` option."**
   ```bash
   kubectl describe configmaps
   ```
   - Read the `Data` section to find the `DB_HOST` key's value.

8. **"Create a new ConfigMap for the `webapp-color` POD."**
   ```bash
   kubectl create configmap webapp-config-map --from-literal=APP_COLOR=darkblue
   ```

9. **"Set the environment option to `envFrom` and use `configMapRef`
   `webapp-config-map`."**
   ```bash
   kubectl get pods webapp-color -o yaml > new-webapp.yaml
   kubectl delete pods webapp-color
   ```
   - Update the Pod definition file — under `spec.containers[]`, add:
     ```yaml
     envFrom:
     - configMapRef:
         name: webapp-config-map
     ```
   ```bash
   kubectl create -f new-webapp.yaml
   ```

10. **"View the changes to the web application UI."**
    - Confirms the app now renders using the ConfigMap-sourced color (`darkblue`) instead
      of a hardcoded literal env var.

**Exam-relevant takeaway:** the lab reinforces the get→delete→edit→recreate cycle for
immutable Pod fields, and the two-step ConfigMap workflow (create the ConfigMap first,
then wire it into the Pod via `envFrom.configMapRef`).

---

*Continued in [kubernetes-application-lifecycle-management-notes_part_02.md](kubernetes-application-lifecycle-management-notes_part_02.md)
— Sections 10–18 (Secrets, Multi-Container Pods, Init Containers, Self-Healing
Applications, Presentation Deck) plus the Quick Revision Checklist.*
