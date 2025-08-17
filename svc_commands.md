# 🌐 Kubernetes Imperative Commands – Services

A **Service** exposes Pods or Deployments to enable stable networking.  
Types: **ClusterIP (default)**, **NodePort**, **LoadBalancer**, **ExternalName**.

---

## 📋 SERVICE COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create service clusterip <name> --tcp=<port>:<targetPort>` | Create ClusterIP Service | `kubectl create service clusterip web --tcp=80:8080` |
| `kubectl create service nodeport <name> --tcp=<port>:<targetPort>` | Create NodePort Service | `kubectl create service nodeport web --tcp=80:8080` |
| `kubectl create service loadbalancer <name> --tcp=<port>:<targetPort>` | Create LoadBalancer Service (cloud only) | `kubectl create service loadbalancer web --tcp=80:8080` |
| `kubectl expose pod <pod> --port=<port> --target-port=<port>` | Expose Pod as Service | `kubectl expose pod nginx --port=80 --target-port=8080` |
| `kubectl expose deployment <name> --port=<port> --target-port=<port>` | Expose Deployment as Service | `kubectl expose deployment web --port=80 --target-port=8080` |
| `kubectl expose replicaset <name> --port=<port>` | Expose ReplicaSet as Service | `kubectl expose rs myrs --port=80` |
| `kubectl expose deployment <name> --port=<port> --type=NodePort` | Expose Deployment externally via NodePort | `kubectl expose deployment web --port=80 --type=NodePort` |
| `kubectl expose deployment <name> --port=<port> --type=LoadBalancer` | Expose Deployment externally via LoadBalancer | `kubectl expose deployment web --port=80 --type=LoadBalancer` |
| `kubectl get services` | List all Services | `kubectl get svc` |
| `kubectl get svc <name>` | Get specific Service | `kubectl get svc web` |
| `kubectl describe svc <name>` | Detailed Service info | `kubectl describe svc web` |
| `kubectl delete svc <name>` | Delete Service | `kubectl delete svc web` |
| `kubectl edit svc <name>` | Edit Service definition | `kubectl edit svc web` |
| `kubectl patch svc <name>` | Patch Service inline | `kubectl patch svc web -p '{"spec":{"type":"NodePort"}}'` |
| `kubectl get endpoints <svc>` | View Service Endpoints | `kubectl get endpoints web` |
| `kubectl port-forward svc/<name> <local>:<remote>` | Forward local port to Service | `kubectl port-forward svc/web 8080:80` |

---

## 📌 NOTES
- **ClusterIP** (default): Accessible inside cluster only.  
- **NodePort**: Accessible on `<NodeIP>:<NodePort>`.  
- **LoadBalancer**: Externally accessible (cloud provider dependent).  
- **ExternalName**: Maps a Service to external DNS (CNAME).  
- `kubectl create service` is shorthand for quickly making **ClusterIP/NodePort/LoadBalancer**.  
- `kubectl expose` is more flexible (can expose Pod, RS, or Deployment).  

---
