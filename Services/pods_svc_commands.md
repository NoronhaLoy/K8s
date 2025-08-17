# 🟢 Kubernetes Imperative Commands – Pods & Services

This cheat sheet lists **all imperative commands for Pods and Services** in tabular format for quick reference.

---

## 🚀 POD COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl run <pod> --image=<img>` | Create a Pod | `kubectl run mypod --image=nginx` |
| `--port=<port>` | Create Pod with a custom port | `kubectl run mypod --image=nginx --port=8080` |
| `--labels` | Create Pod with labels | `kubectl run mypod --image=nginx --labels="env=dev,app=web"` |
| `--env` | Create Pod with env variables | `kubectl run mypod --image=nginx --env="ENV=prod" --env="VERSION=1.0"` |
| `--command -- <cmd>` | Override default command | `kubectl run mypod --image=busybox --command -- sleep 3600` |
| `-n <namespace>` | Create Pod in specific namespace | `kubectl run mypod --image=nginx -n test-ns` |
| `--dry-run=client -o yaml` | Generate Pod YAML (for editing) | `kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml` |
| `kubectl get pods` | List Pods | `kubectl get pods -A` |
| `kubectl get pod <name>` | Get Pod details | `kubectl get pod mypod -o wide` |
| `kubectl get pods -l <label>` | List Pods by label | `kubectl get pods -l app=web` |
| `kubectl describe pod <name>` | Show Pod details | `kubectl describe pod mypod` |
| `kubectl delete pod <name>` | Delete Pod | `kubectl delete pod mypod` |
| `kubectl delete pods -l <label>` | Delete Pods by label | `kubectl delete pods -l app=web` |
| `kubectl delete pod <name> --grace-period=0 --force` | Force delete Pod | `kubectl delete pod mypod --grace-period=0 --force` |
| `kubectl exec -it <pod> -- <cmd>` | Execute command in Pod | `kubectl exec -it mypod -- /bin/bash` |
| `kubectl exec -it <pod> -c <container>` | Exec into specific container | `kubectl exec -it mypod -c sidecar -- sh` |
| `kubectl cp <pod>:<path> <local>` | Copy from Pod to local | `kubectl cp mypod:/etc/nginx/nginx.conf ./` |
| `kubectl cp <local> <pod>:<path>` | Copy from local to Pod | `kubectl cp ./index.html mypod:/usr/share/nginx/html/` |
| `kubectl logs <pod>` | Get Pod logs | `kubectl logs mypod` |
| `kubectl logs -f <pod>` | Stream logs | `kubectl logs -f mypod` |
| `kubectl logs <pod> -c <container>` | Logs of container in Pod | `kubectl logs mypod -c sidecar` |
| `kubectl logs --since=1h --tail=100 <pod>` | Filter logs | `kubectl logs --since=1h --tail=100 mypod` |
| `kubectl port-forward pod/<pod> <local>:<pod-port>` | Forward local port to Pod | `kubectl port-forward pod/mypod 8080:80` |
| `kubectl label pod <pod>` | Add label to Pod | `kubectl label pod mypod env=prod --overwrite` |
| `kubectl annotate pod <pod>` | Add annotation to Pod | `kubectl annotate pod mypod owner="team-a" --overwrite` |
| `kubectl set image pod/<pod>` | Update Pod image | `kubectl set image pod/mypod nginx=nginx:1.27.0` |
| `kubectl top pod <pod>` | Show Pod resource usage | `kubectl top pod mypod` |

---

## 🌐 SERVICE COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl expose pod <pod>` | Expose Pod as ClusterIP (default) | `kubectl expose pod mypod --port=80 --target-port=8080 --name=mypod-svc` |
| `kubectl expose pod <pod> --type=NodePort` | Expose Pod as NodePort | `kubectl expose pod mypod --type=NodePort --port=80 --target-port=8080 --name=mypod-np` |
| `kubectl expose deployment <deploy>` | Expose Deployment | `kubectl expose deployment mydeploy --port=80 --target-port=8080 --name=mydeploy-svc` |
| `kubectl expose deployment <deploy> --type=LoadBalancer` | Expose Deployment as LoadBalancer | `kubectl expose deployment mydeploy --type=LoadBalancer --port=80 --target-port=8080 --name=mydeploy-lb` |
| `kubectl get svc` | List Services | `kubectl get svc -A` |
| `kubectl get svc <name>` | Get Service details | `kubectl get svc mypod-svc -o wide` |
| `kubectl describe svc <name>` | Show Service details | `kubectl describe svc mypod-svc` |
| `kubectl port-forward svc/<svc> <local>:<svc-port>` | Forward port to Service | `kubectl port-forward svc/mypod-svc 8080:80` |
| `kubectl edit svc <svc>` | Edit Service in editor | `kubectl edit svc mypod-svc` |
| `kubectl patch svc <svc>` | Patch Service type/values | `kubectl patch svc mypod-svc -p '{"spec": {"type": "NodePort"}}'` |
| `kubectl delete svc <svc>` | Delete Service | `kubectl delete svc mypod-svc` |
| `kubectl delete svc -l <label>` | Delete Services by label | `kubectl delete svc -l app=web` |

---

## 📌 NOTES

- **ClusterIP** → Default, internal only.  
- **NodePort** → Exposed via node IP + static port.  
- **LoadBalancer** → External via cloud LB.  
- Pods are usually managed by **Deployments, ReplicaSets, Jobs**.  
- Use **imperative commands** for quick testing/debugging, but prefer **YAML manifests** for production.

---
