# 🗂️ Kubernetes Imperative Commands – Remaining Resources

This section covers remaining Kubernetes objects not included earlier:  
**Jobs, CronJobs, Namespaces, Nodes, RBAC, DaemonSets, HPA, Quotas, LimitRanges, NetworkPolicies.**

---

## 📌 Jobs
A **Job** creates one or more Pods to run a task until completion.

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create job <name> --image=<image> -- <cmd>` | Create Job | `kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'` |
| `kubectl get jobs` | List Jobs | `kubectl get jobs` |
| `kubectl describe job <name>` | Describe Job | `kubectl describe job pi` |
| `kubectl delete job <name>` | Delete Job | `kubectl delete job pi` |
| `kubectl logs job/<name>` | View logs of Job Pods | `kubectl logs job/pi` |
| `kubectl create job --from=cronjob/<cj>` | Create Job from CronJob | `kubectl create job test --from=cronjob/my-cron` |

---

## 📌 CronJobs
A **CronJob** runs Jobs on a **schedule** (like cron in Linux).

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create cronjob <name> --image=<image> --schedule="<cron>" -- <cmd>` | Create CronJob | `kubectl create cronjob hello --image=busybox --schedule="*/1 * * * *" -- echo hello` |
| `kubectl get cronjobs` | List CronJobs | `kubectl get cronjobs` |
| `kubectl describe cronjob <name>` | Show details | `kubectl describe cronjob hello` |
| `kubectl delete cronjob <name>` | Delete CronJob | `kubectl delete cronjob hello` |

---

## 📌 Namespaces
**Namespaces** provide isolation between Kubernetes resources.

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create namespace <name>` | Create Namespace | `kubectl create namespace dev` |
| `kubectl get namespaces` | List all namespaces | `kubectl get ns` |
| `kubectl delete namespace <name>` | Delete Namespace | `kubectl delete ns dev` |
| `kubectl config set-context --current --namespace=<name>` | Switch Namespace | `kubectl config set-context --current --namespace=dev` |

---

## 📌 Nodes
**Nodes** are worker machines in Kubernetes that run Pods.

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl get nodes` | List all nodes | `kubectl get nodes` |
| `kubectl describe node <node>` | Show details of a node | `kubectl describe node worker-1` |
| `kubectl cordon <node>` | Mark node unschedulable | `kubectl cordon worker-1` |
| `kubectl uncordon <node>` | Mark node schedulable | `kubectl uncordon worker-1` |
| `kubectl drain <node> --ignore-daemonsets` | Evict Pods and prepare for maintenance | `kubectl drain worker-1 --ignore-daemonsets` |

---

## 📌 RBAC (Roles & RoleBindings)
**RBAC (Role-Based Access Control)** manages access to resources.

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create role <name> --verb=<verbs> --resource=<resources>` | Create Role | `kubectl create role pod-reader --verb=get,list,watch --resource=pods` |
| `kubectl create rolebinding <name> --role=<role> --user=<user>` | Bind Role to User | `kubectl create rolebinding read-pods --role=pod-reader --user=dev-user` |
| `kubectl create clusterrole <name> --verb=<verbs> --resource=<resources>` | Create ClusterRole | `kubectl create clusterrole pod-admin --verb=* --resource=pods` |
| `kubectl create clusterrolebinding <name> --clusterrole=<cr> --user=<user>` | Bind ClusterRole to User | `kubectl create clusterrolebinding admin-binding --clusterrole=pod-admin --user=dev-user` |
| `kubectl create serviceaccount <name>` | Create ServiceAccount | `kubectl create serviceaccount my-sa` |

---

## 📌 DaemonSets
**DaemonSets** ensure a Pod runs on **every node** (commonly used for monitoring, logging, networking).

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create daemonset <name> --image=<image>` | Create DaemonSet | `kubectl create daemonset logger --image=fluentd` |
| `kubectl get daemonsets` | List DaemonSets | `kubectl get daemonsets` |
| `kubectl describe daemonset <name>` | Show details | `kubectl describe daemonset logger` |
| `kubectl delete daemonset <name>` | Delete DaemonSet | `kubectl delete daemonset logger` |

---

## 📌 Horizontal Pod Autoscaler (HPA)
**HPA** automatically scales Pods based on CPU/memory usage.

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl autoscale deployment <name> --cpu-percent=<n> --min=<n> --max=<n>` | Create HPA | `kubectl autoscale deployment web --cpu-percent=80 --min=2 --max=5` |
| `kubectl get hpa` | List HPAs | `kubectl get hpa` |
| `kubectl describe hpa <name>` | Show details | `kubectl describe hpa web` |
| `kubectl delete hpa <name>` | Delete HPA | `kubectl delete hpa web` |

---

## 📌 Resource Quotas
**ResourceQuotas** limit resource usage within a namespace.

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create quota <name> --hard=cpu=<n>,memory=<mem>,pods=<n>` | Create ResourceQuota | `kubectl create quota dev-quota --hard=cpu=2,memory=1Gi,pods=10` |
| `kubectl get quota` | List quotas | `kubectl get quota` |
| `kubectl describe quota <name>` | Show details | `kubectl describe quota dev-quota` |
| `kubectl delete quota <name>` | Delete quota | `kubectl delete quota dev-quota` |

---

## 📌 LimitRanges
**LimitRanges** set default CPU/memory limits for Pods/Containers in a namespace.

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create -f limitrange.yaml` | Create LimitRange from YAML | `kubectl create -f limitrange.yaml` |
| `kubectl get limitrange` | List LimitRanges | `kubectl get limitrange` |
| `kubectl describe limitrange <name>` | Show details | `kubectl describe limitrange mem-limit-range` |
| `kubectl delete limitrange <name>` | Delete LimitRange | `kubectl delete limitrange mem-limit-range` |

---

## 📌 NetworkPolicies
**NetworkPolicies** control how Pods communicate with each other and other network endpoints.  
⚠️ Must be created via YAML (imperative commands are very limited).

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create -f networkpolicy.yaml` | Create NetworkPolicy | `kubectl create -f allow-db.yaml` |
| `kubectl get networkpolicy` | List NetworkPolicies | `kubectl get networkpolicy` |
| `kubectl describe networkpolicy <name>` | Show details | `kubectl describe networkpolicy allow-db` |
| `kubectl delete networkpolicy <name>` | Delete NetworkPolicy | `kubectl delete networkpolicy allow-db` |

---

# Kubernetes Imperative Commands (Extended Resources)

---

## PodDisruptionBudget (PDB)
Ensures a minimum number of pods are always running during voluntary disruptions (like node drain).

| Command | Example | Description |
|---------|---------|-------------|
| Create PDB | `kubectl create pdb my-pdb --selector=app=myapp --min-available=2` | Ensures at least 2 pods remain available during disruptions |
| Get PDBs | `kubectl get pdb` | List all PodDisruptionBudgets |
| Describe PDB | `kubectl describe pdb my-pdb` | Show details of a PodDisruptionBudget |
| Delete PDB | `kubectl delete pdb my-pdb` | Remove a PodDisruptionBudget |

---

## PriorityClass
Defines scheduling priority for pods.

| Command | Example | Description |
|---------|---------|-------------|
| Create PriorityClass | `kubectl create priorityclass high-priority --value=1000 --description="High priority" --global-default=false` | Create a new PriorityClass |
| Get PriorityClasses | `kubectl get priorityclass` | List all PriorityClasses |
| Delete PriorityClass | `kubectl delete priorityclass high-priority` | Remove a PriorityClass |

---

## Endpoints
Manually define network endpoints for a Service (usually created automatically).

| Command | Example | Description |
|---------|---------|-------------|
| Create Endpoints | `kubectl create endpoints my-svc --address=10.1.1.5 --port=9376` | Create an endpoint resource manually |
| Get Endpoints | `kubectl get endpoints` | List all endpoints |
| Describe Endpoints | `kubectl describe endpoints my-svc` | Show details of an endpoint |
| Delete Endpoints | `kubectl delete endpoints my-svc` | Remove endpoint |

---

## CertificateSigningRequest (CSR)
Used to request and approve TLS certificates in Kubernetes.

| Command | Example | Description |
|---------|---------|-------------|
| Create CSR | `kubectl create -f csr.yaml` | Create a CSR from a manifest |
| Get CSRs | `kubectl get csr` | List all CSRs |
| Approve CSR | `kubectl certificate approve my-csr` | Approve a CSR request |
| Deny CSR | `kubectl certificate deny my-csr` | Deny a CSR request |
| Delete CSR | `kubectl delete csr my-csr` | Remove a CSR |

---

## CustomResourceDefinition (CRD)
Extends Kubernetes API with custom objects.

| Command | Example | Description |
|---------|---------|-------------|
| Create CRD | `kubectl create crd myresources.example.com --group=example.com --version=v1 --names=kind=MyResource,singular=myresource,plural=myresources` | Create a custom resource definition |
| Get CRDs | `kubectl get crd` | List all CRDs |
| Describe CRD | `kubectl describe crd myresources.example.com` | Show details of a CRD |
| Delete CRD | `kubectl delete crd myresources.example.com` | Remove a CRD |

---

## Events
Events provide insight into what’s happening in the cluster.

| Command | Example | Description |
|---------|---------|-------------|
| Get Events | `kubectl get events` | List all events in default namespace |
| Sort Events by Time | `kubectl get events --sort-by=.metadata.creationTimestamp` | Show events in chronological order |
| Describe Events | `kubectl describe events` | Show detailed events info |

---
# Kubernetes Imperative Commands (Final Additions)

---

## EndpointSlice
Represents groups of network endpoints (modern replacement for Endpoints, usually auto-managed).

| Command | Example | Description |
|---------|---------|-------------|
| Get EndpointSlices | `kubectl get endpointslice` | List all EndpointSlices |
| Describe EndpointSlice | `kubectl describe endpointslice my-svc-abc123` | Show details of an EndpointSlice |
| Delete EndpointSlice | `kubectl delete endpointslice my-svc-abc123` | Remove an EndpointSlice (usually recreated automatically) |

---

## VolumeSnapshot (CSI-based)
Used to create snapshots of PersistentVolumes (requires CSI snapshot controller).

| Command | Example | Description |
|---------|---------|-------------|
| Create Snapshot | `kubectl create -f snapshot.yaml` | Create a VolumeSnapshot from YAML |
| Get Snapshots | `kubectl get volumesnapshot` | List all VolumeSnapshots |
| Describe Snapshot | `kubectl describe volumesnapshot my-snap` | Show details of a snapshot |
| Delete Snapshot | `kubectl delete volumesnapshot my-snap` | Delete a snapshot |

---

## Logs & Debugging
Imperative commands to inspect pod logs and debug containers.

| Command | Example | Description |
|---------|---------|-------------|
| View Pod Logs | `kubectl logs my-pod` | Show logs of a pod |
| View Logs of Specific Container | `kubectl logs my-pod -c my-container` | Show logs for a container inside a pod |
| Stream Logs | `kubectl logs -f my-pod` | Stream live logs |
| Previous Logs | `kubectl logs my-pod --previous` | Show logs from a previously crashed container |
| Run Command in Pod | `kubectl exec -it my-pod -- /bin/sh` | Open shell inside a running pod |
| Copy Files From Pod | `kubectl cp my-pod:/path/in/pod ./localpath` | Copy file from pod to local |
| Copy Files To Pod | `kubectl cp ./localpath my-pod:/path/in/pod` | Copy file from local to pod |

---

## Metrics
Requires Metrics Server installed in the cluster.

| Command | Example | Description |
|---------|---------|-------------|
| Get Node Metrics | `kubectl top nodes` | Show CPU & memory usage of nodes |
| Get Pod Metrics | `kubectl top pods` | Show CPU & memory usage of pods |
| Get Pod Metrics by Namespace | `kubectl top pods -n kube-system` | Show metrics for specific namespace |

---

## Affinity & Anti-Affinity
Scheduling rules to influence pod placement (only via YAML, not imperative).  
You can **generate a skeleton pod with affinity** using:

| Command | Example | Description |
|---------|---------|-------------|
| Generate Pod with Affinity | `kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml` | Create base YAML and edit with affinity/anti-affinity rules |
| Apply | `kubectl apply -f pod.yaml` | Apply pod with custom affinity rules |

---

## TopologySpreadConstraints
Distribute pods evenly across topology domains (nodes, zones, regions).

| Command | Example | Description |
|---------|---------|-------------|
| Generate Pod | `kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml` | Create pod YAML and add topologySpreadConstraints section manually |
| Apply | `kubectl apply -f pod.yaml` | Apply pod with spread constraints |

---

## Webhook Configurations
Admission controllers for mutating/validating requests. Advanced use case, typically YAML-based.

| Command | Example | Description |
|---------|---------|-------------|
| Create ValidatingWebhook | `kubectl create -f validating-webhook.yaml` | Create a validating webhook |
| Get Webhooks | `kubectl get validatingwebhookconfigurations` | List validating webhooks |
| Delete Webhook | `kubectl delete validatingwebhookconfiguration my-webhook` | Remove a webhook |
| MutatingWebhook | `kubectl get mutatingwebhookconfigurations` | List mutating webhooks |

---



