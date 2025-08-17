# 🐳 Kubernetes Imperative Commands – Pods

A **Pod** is the smallest deployable unit in Kubernetes.  
Imperative commands let you create, view, debug, and delete Pods directly.

---

## 📋 POD COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl run <name> --image=<image>` | Create a Pod (quick way) | `kubectl run nginx --image=nginx` |
| `kubectl run <name> --image=<image> --restart=Never` | Create a Pod (not Deployment) | `kubectl run mypod --image=busybox --restart=Never` |
| `kubectl run <name> --image=<image> --dry-run=client -o yaml` | Generate Pod manifest (YAML) without creating | `kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml` |
| `kubectl create -f pod.yaml` | Create Pod from definition file | `kubectl create -f pod.yaml` |
| `kubectl expose pod <pod> --port=<port> --target-port=<port>` | Expose Pod as Service | `kubectl expose pod nginx --port=80 --target-port=8080` |
| `kubectl get pods` | List all Pods | `kubectl get pods` |
| `kubectl get pod <name>` | Get specific Pod | `kubectl get pod nginx` |
| `kubectl describe pod <name>` | Detailed Pod info (events, containers, status) | `kubectl describe pod nginx` |
| `kubectl delete pod <name>` | Delete Pod | `kubectl delete pod nginx` |
| `kubectl delete -f pod.yaml` | Delete Pod from file | `kubectl delete -f pod.yaml` |
| `kubectl exec -it <pod> -- <command>` | Run a command in Pod container | `kubectl exec -it nginx -- /bin/bash` |
| `kubectl logs <pod>` | Get logs from Pod | `kubectl logs nginx` |
| `kubectl logs -f <pod>` | Stream Pod logs | `kubectl logs -f nginx` |
| `kubectl cp <pod>:<path> <local-path>` | Copy files from Pod | `kubectl cp nginx:/var/log/nginx ./nginx-logs` |
| `kubectl attach <pod>` | Attach to a running Pod | `kubectl attach -it nginx` |
| `kubectl port-forward pod/<name> <local>:<remote>` | Forward local port to Pod | `kubectl port-forward pod/nginx 8080:80` |
| `kubectl label pod <name> <key>=<value>` | Add label to Pod | `kubectl label pod nginx app=web` |
| `kubectl annotate pod <name> <key>=<value>` | Add annotation to Pod | `kubectl annotate pod nginx description='frontend pod'` |
| `kubectl edit pod <name>` | Edit Pod definition | `kubectl edit pod nginx` |
| `kubectl patch pod <name>` | Patch Pod inline | `kubectl patch pod nginx -p '{"metadata":{"labels":{"env":"prod"}}}'` |

---

## 📌 NOTES
- `kubectl run` with `--restart=Never` creates a **Pod**.  
- Without `--restart=Never`, `kubectl run` creates a **Deployment** instead.  
- `kubectl create -f file.yaml` is for creating Pods from YAML.  
- `kubectl expose` turns Pods into **Services**.  
- For debugging: `exec`, `logs`, `port-forward`, and `cp` are most useful.  

---
