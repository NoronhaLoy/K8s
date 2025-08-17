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
