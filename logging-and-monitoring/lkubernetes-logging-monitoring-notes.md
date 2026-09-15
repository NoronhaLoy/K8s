# Kubernetes Logging & Monitoring — Complete Notes

> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/04-Logging-and-Monitoring/`
> Covers: Section Introduction, Monitor Cluster Components, Practice Test (Monitor Cluster
> Components), Managing Application Logs, Presentation Deck, Practice Test (Managing
> Application Logs).

---

## 01. Logging and Monitoring — Section Introduction

- This section of the CKA course is broken into four themes:
  - Monitor Cluster Components
  - Monitor Applications
  - Monitor Cluster Components Logs
  - Application Logs
- Video reference: *Logging and Monitoring Section Introduction* (KodeKloud).
- Big picture: "monitoring" = numeric resource metrics (CPU/memory usage over time);
  "logging" = the actual stdout/stderr text streams emitted by containers and cluster
  components — two different problems solved with different tooling in Kubernetes.

---

## 02. Monitor Cluster Components

### What do you actually want to monitor?

- Node-level metrics:
  - Number of nodes in the cluster
  - Overall resource consumption per node (CPU, memory, disk, network)
- Pod-level metrics:
  - Performance of each Pod (CPU/memory usage per Pod/container)
- Kubernetes does **not** come with a full-featured built-in monitoring solution out of
  the box — you must deploy something.

### Third-party / full monitoring solutions

- **Metrics Server** — lightweight, in-memory, cluster-wide metrics utility (the one this
  course focuses on for the exam).
- Other options mentioned in the ecosystem (fuller-featured, persist historical data):
  - Prometheus
  - Elastic Stack
  - Datadog
  - Dynatrace

### Heapster vs Metrics Server

- **Heapster** was the original Kubernetes monitoring/metrics-collection add-on.
- Heapster is now **deprecated** — a slimmed-down fork/replacement was created called the
  **Metrics Server**.
- Metrics Server is the modern, minimal, in-memory alternative that the Kubernetes project
  now maintains and recommends.

### Metrics Server — how it works

- **One Metrics Server per cluster** — a single Metrics Server instance is enough to serve
  the whole cluster's metrics.
- Metrics Server retrieves metrics from each node and pod, **aggregates them in memory**.
  - Key implication: Metrics Server does **not** store metrics on disk — it holds only
    in-memory, near-real-time data.
  - You therefore cannot see historical trends/history from Metrics Server alone; for
    historical data you need one of the full monitoring solutions (Prometheus, Elastic
    Stack, etc.).
- **How metrics for Pods on nodes are generated:**
  - The **Kubelet** on each node contains a subcomponent called **cAdvisor** (Container
    Advisor).
  - cAdvisor is responsible for retrieving performance metrics from pods and exposing them
    via the Kubelet API, so that they're available to the Metrics Server (or Heapster,
    historically).

### Metrics Server — Getting Started (setup steps)

1. Clone the metrics-server repo from GitHub:
   ```bash
   git clone https://github.com/kubernetes-incubator/metrics-server.git
   ```
2. Deploy the metrics-server components:
   ```bash
   kubectl create -f metric-server/deploy/1.8+/
   ```
3. View cluster-wide node performance:
   ```bash
   kubectl top node
   ```
4. View Pod-level performance metrics:
   ```bash
   kubectl top pod
   ```
- `kubectl top node` and `kubectl top pod` are the two commands built on top of the
  Metrics Server API (`metrics.k8s.io`) — they are the primary exam-relevant commands
  for this topic.
- On managed/minikube setups, the actual repo to deploy from in later course material is
  KodeKloud's own fork (see Practice Test below) rather than the original
  kubernetes-incubator repo, since the upstream repo has moved/been archived over time.

---

## 03. Practice Test — Monitor Cluster Components

Step-by-step solution walkthrough used in the hands-on lab:

1. **Inspect existing workloads**
   ```bash
   kubectl get pods
   ```
2. **Clone the metrics-server deployment repo** (practice-test uses KodeKloud's maintained
   fork, not the original archived kubernetes-incubator one):
   ```bash
   git clone https://github.com/kodekloudhub/kubernetes-metrics-server.git
   ```
3. **Deploy metrics-server** by creating all downloaded components:
   ```bash
   cd kubernetes-metrics-server
   kubectl create -f .
   ```
4. **Wait for metrics to populate** — Metrics Server takes a few minutes after deployment
   before `kubectl top` returns valid data (it needs at least one scrape interval to have
   passed).
   ```bash
   kubectl top node
   ```
5. **Identify the node consuming the most CPU (cores)**
   ```bash
   kubectl top node
   ```
   - Read the `CPU(cores)` column of the output.
6. **Identify the node consuming the most Memory (bytes)**
   ```bash
   kubectl top node
   ```
   - Read the `MEMORY(bytes)` column of the output.
7. **Identify the Pod consuming the most Memory (bytes)**
   ```bash
   kubectl top pod
   ```
   - Read the `MEMORY(bytes)` column.
8. **Identify the Pod consuming the least CPU (cores)**
   ```bash
   kubectl top pod
   ```
   - Read the `CPU(cores)` column, find the lowest value.

**Exam-relevant takeaway:** all of these questions reduce to "run `kubectl top node` or
`kubectl top pod`, then read the correct column" — no YAML editing is required, but
metrics-server **must be running** first, or the commands return an error like
`error: Metrics API not available`.

---

## 04. Managing Application Logs

### Logging in Docker (background/refresher)

- A single Docker container running an app (e.g., an event-simulator writing timestamped
  log lines to stdout) can have its logs viewed with:
  ```bash
  docker run -d kodekloud/event-simulator
  docker logs -f <container-id>
  ```
  - `-f` follows/streams the logs live, same idea as `tail -f`.
- When you run **multiple related containers** with `docker-compose` (e.g., an
  event-simulator plus a separate "message handler" container), each service gets its own
  log stream, and `docker logs <container-name-or-id>` is scoped to just that one
  container — this sets up *why* Kubernetes needs a `<container-name>` argument once a Pod
  has more than one container (see below).

### Logs in Kubernetes

- Example single-container Pod definition used for the demo:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: event-simulator-pod
  spec:
    containers:
    - name: event-simulator
      image: kodekloud/event-simulator
  ```
- **View logs of a Pod (single container):**
  ```bash
  kubectl logs -f event-simulator-pod
  ```
  - `-f` streams/follows logs in real time, same as Docker.
- **Multi-container Pods — container name is required:**
  - If a Pod runs more than one container, `kubectl logs` cannot infer which container's
    logs you want, so you **must** pass the container name explicitly:
    ```bash
    kubectl logs -f <pod-name> <container-name>
    kubectl logs -f event-simulator-pod event-simulator
    ```
  - Omitting the container name on a multi-container Pod results in an error asking you to
    specify one.

### Key command variants worth remembering (exam scope)

- `kubectl logs <pod>` — dump current logs, no follow.
- `kubectl logs -f <pod>` — stream/follow logs live.
- `kubectl logs <pod> <container>` — required once a Pod has 2+ containers.
- `kubectl logs --previous <pod>` — view logs from a previous (crashed/restarted)
  instance of the container (useful for `CrashLoopBackOff` debugging — implied context
  from the broader logging topic even though not shown verbatim in the source doc).

### Reference

- Kubernetes blog: [Cluster-Level Logging with Kubernetes](https://kubernetes.io/blog/2015/06/cluster-level-logging-with-kubernetes/)
  — background on why cluster-level logging (aggregating logs across all nodes/pods, e.g.
  via a sidecar/agent + backend like Elasticsearch/Fluentd/Kibana) is a separate, harder
  problem than just reading one Pod's logs with `kubectl logs`.

---

## 05. Download Presentation Deck

- The section provides a downloadable slide deck accompanying the video lectures:
  [Presentation Deck](https://kodekloud.com/topic/download-presentation-deck-3/).
- Purely a resource/reference link — no technical content of its own.

---

## 06. Practice Test — Managing Application Logs

Step-by-step solution walkthrough:

1. **Inspect a deployed Pod hosting an application; wait for it to start.**
   ```bash
   kubectl get pods
   ```
2. **Inspect the logs of that Pod (`webapp-1`).**
   ```bash
   kubectl logs webapp-1
   ```
3. **A second Pod (`webapp-2`) is deployed; inspect it and wait for it to start.**
   ```bash
   kubectl get pods
   ```
4. **Inspect the logs of the webapp running in `webapp-2`.**
   ```bash
   kubectl logs webapp-2
   ```

**Pattern to remember:** every "inspect the logs" task in the lab is solved by the same
two-step reflex — `kubectl get pods` to confirm the Pod exists/is Running, then
`kubectl logs <pod-name>` (adding the container name only if the Pod has multiple
containers).

---

## Quick Revision Checklist

- [ ] Know that Kubernetes ships **no built-in monitoring**; Metrics Server (or
      Prometheus/Elastic Stack/Datadog/Dynatrace) must be deployed separately.
- [ ] Know Heapster is deprecated → replaced by Metrics Server.
- [ ] Understand Metrics Server stores metrics **in-memory only** (no history) — one
      instance per cluster.
- [ ] Know that **cAdvisor**, embedded in the **Kubelet**, is what actually gathers
      per-Pod resource metrics on each node.
- [ ] Can run and interpret `kubectl top node` and `kubectl top pod` (columns:
      `CPU(cores)`, `MEMORY(bytes)`).
- [ ] Remember Metrics Server needs a few minutes after deployment before `kubectl top`
      returns data.
- [ ] Know `kubectl logs -f <pod>` for single-container Pods.
- [ ] Know `kubectl logs -f <pod> <container>` is **required** (not optional) once a Pod
      has multiple containers.
- [ ] Know `kubectl logs --previous <pod>` for viewing logs from a crashed/restarted
      container.

---

*End of Logging & Monitoring notes.*
