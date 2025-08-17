# 🚀 Kubernetes Imperative Commands – Deployments

A **Deployment** manages ReplicaSets and Pods, allowing rolling updates, rollbacks, and scaling.

---

## 📋 DEPLOYMENT COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create deployment <name> --image=<image>` | Create a Deployment | `kubectl create deployment web --image=nginx` |
| `kubectl create deployment <name> --image=<image> --replicas=<n>` | Create Deployment with replicas | `kubectl create deployment web --image=nginx --replicas=3` |
| `kubectl run <name> --image=<image> --restart=Always` | Alternative way to create a Deployment | `kubectl run web --image=nginx --restart=Always` |
| `kubectl get deployments` | List all Deployments | `kubectl get deployments` |
| `kubectl get deployment <name>` | Get specific Deployment | `kubectl get deployment web -o wide` |
| `kubectl describe deployment <name>` | Detailed info of Deployment | `kubectl describe deployment web` |
| `kubectl delete deployment <name>` | Delete Deployment | `kubectl delete deployment web` |
| `kubectl delete deployment -l <label>` | Delete Deployment by label | `kubectl delete deployment -l app=web` |
| `kubectl edit deployment <name>` | Edit Deployment definition | `kubectl edit deployment web` |
| `kubectl scale deployment <name> --replicas=<n>` | Scale Deployment | `kubectl scale deployment web --replicas=5` |
| `kubectl rollout status deployment/<name>` | Check rollout status | `kubectl rollout status deployment/web` |
| `kubectl rollout history deployment/<name>` | View rollout history | `kubectl rollout history deployment/web` |
| `kubectl rollout undo deployment/<name>` | Rollback to previous version | `kubectl rollout undo deployment/web` |
| `kubectl set image deployment/<name> <container>=<image>` | Update container image | `kubectl set image deployment/web nginx=nginx:1.27.0` |
| `kubectl annotate deployment <name> key=value` | Add annotation | `kubectl annotate deployment web owner="team-a"` |
| `kubectl label deployment <name> key=value` | Add/update label | `kubectl label deployment web env=prod --overwrite` |
| `kubectl expose deployment <name>` | Expose Deployment as Service | `kubectl expose deployment web --port=80 --target-port=8080 --type=NodePort` |
| `kubectl patch deployment <name>` | Patch Deployment inline | `kubectl patch deployment web -p '{"spec":{"replicas":4}}'` |
| `kubectl get pods -l <label>` | Get Pods created by Deployment | `kubectl get pods -l app=web` |

---

## 📌 NOTES
- **Deployment → ReplicaSet → Pods** (chain of ownership).  
- Recommended way to manage apps instead of raw ReplicaSets.  
- Supports **rolling updates & rollback** (not possible with ReplicaSets alone).  
- If you update Pod specs inside a Deployment → a **new ReplicaSet** is created automatically.  

---
