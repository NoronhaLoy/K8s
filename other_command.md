# 🏗️ Kubernetes Imperative Commands – Remaining Objects

---

## 🕒 JOBS & CRONJOBS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create job <name> --image=<image> -- <command>` | Create a one-time Job | `kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'` |
| `kubectl create job <name> --image=<image> --dry-run=client -o yaml > job.yaml` | Generate YAML for Job | `kubectl create job pi --image=perl --dry-run=client -o yaml > job.yaml` |
| `kubectl create cronjob <name> --image=<image> --schedule="<cron>" -- <command>` | Create a CronJob | `kubectl create cronjob hello --image=busybox --schedule="*/1 * * * *" -- echo "hello"` |
| `kubectl get jobs` | List Jobs | `kubectl get jobs` |
| `kubectl get cronjob` | List CronJobs | `kubectl get cronjob` |
| `kubectl describe job <name>` | Describe a Job | `kubectl describe job pi` |
| `kubectl delete job <name>` | Delete a Job | `kubectl delete job pi` |
| `kubectl delete cronjob <name>` | Delete a CronJob | `kubectl delete cronjob hello` |

---

## 🏷️ NAMESPACES

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create namespace <name>` | Create a namespace | `kubectl create namespace dev` |
| `kubectl get ns` | List namespaces | `kubectl get ns` |
| `kubectl describe ns <name>` | Describe a namespace | `kubectl describe ns dev` |
| `kubectl delete ns <name>` | Delete a namespace | `kubectl delete ns dev` |
| `kubectl get pods -n <namespace>` | List pods in a namespace | `kubectl get pods -n dev` |
| `kubectl config set-context --current --namespace=<name>` | Set default namespace | `kubectl config set-context --current --namespace=dev` |

---

## 🔑 RBAC (ROLES, ROLEBINDINGS, SERVICEACCOUNTS)

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create serviceaccount <name>` | Create a service account | `kubectl create serviceaccount my-sa` |
| `kubectl get serviceaccounts` | List service accounts | `kubectl get sa` |
| `kubectl create role <name> --verb=get,list,watch --resource=pods` | Create a role | `kubectl create role pod-reader --verb=get,list,watch --resource=pods` |
| `kubectl create rolebinding <name> --role=<role> --serviceaccount=<ns>:<sa>` | Bind role to SA | `kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:my-sa` |
| `kubectl create clusterrole <name> --verb=get,list,watch --resource=pods` | Create cluster role | `kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods` |
| `kubectl create clusterrolebinding <name> --clusterrole=<role> --serviceaccount=<ns>:<sa>` | Bind cluster role | `kubectl create clusterrolebinding read-pods --clusterrole=pod-reader --serviceaccount=default:my-sa` |

---

## ⚙️ DAEMONSETS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create daemonset <name> --image=<image>` | Create a DaemonSet | `kubectl create daemonset nginx-ds --image=nginx` |
| `kubectl get daemonset` | List DaemonSets | `kubectl get ds` |
| `kubectl describe daemonset <name>` | Describe DaemonSet | `kubectl describe ds nginx-ds` |
| `kubectl delete daemonset <name>` | Delete DaemonSet | `kubectl delete ds nginx-ds` |

---

## 📈 HORIZONTAL POD AUTOSCALER (HPA)

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl autoscale deployment <name> --cpu-percent=<value> --min=<min> --max=<max>` | Create an HPA | `kubectl autoscale deployment nginx --cpu-percent=80 --min=2 --max=5` |
| `kubectl get hpa` | List HPAs | `kubectl get hpa` |
| `kubectl describe hpa <name>` | Describe HPA | `kubectl describe hpa nginx` |
| `kubectl delete hpa <name>` | Delete HPA | `kubectl delete hpa nginx` |

---

## 🔒 RESOURCE QUOTAS & LIMITRANGES

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create quota <name> --hard=cpu=2,memory=1Gi,pods=10` | Create a ResourceQuota | `kubectl create quota dev-quota --hard=cpu=2,memory=1Gi,pods=10` |
| `kubectl get quota` | List quotas | `kubectl get quota` |
| `kubectl describe quota <name>` | Describe a quota | `kubectl describe quota dev-quota` |
| `kubectl delete quota <name>` | Delete a quota | `kubectl delete quota dev-quota` |
| `kubectl create -f limitrange.yaml` | Create a LimitRange | `kubectl create -f limitrange.yaml` |
| `kubectl get limitrange` | List LimitRanges | `kubectl get limitrange` |

---

## 🌐 NETWORK POLICIES

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create -f networkpolicy.yaml` | Create NetworkPolicy from file | `kubectl create -f deny-all.yaml` |
| `kubectl get networkpolicy` | List all policies | `kubectl get netpol` |
| `kubectl describe networkpolicy <name>` | Describe a policy | `kubectl describe netpol deny-all` |
| `kubectl delete networkpolicy <name>` | Delete a policy | `kubectl delete netpol deny-all` |

---

## 🖥️ NODES & CLUSTER ADMIN

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl get nodes` | List nodes | `kubectl get nodes` |
| `kubectl describe node <name>` | Describe node | `kubectl describe node worker-1` |
| `kubectl cordon <node>` | Mark node unschedulable | `kubectl cordon worker-1` |
| `kubectl uncordon <node>` | Mark node schedulable | `kubectl uncordon worker-1` |
| `kubectl drain <node> --ignore-daemonsets` | Drain node for maintenance | `kubectl drain worker-1 --ignore-daemonsets` |
| `kubectl top nodes` | Show node resource usage (metrics-server required) | `kubectl top nodes` |

---

