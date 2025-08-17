# 🔁 Kubernetes Imperative Commands – ReplicaSets

This cheat sheet lists **imperative commands for ReplicaSets** in tabular format.

---

## 🚀 REPLICASETS COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create replicaset -f <file>.yaml` | Create a ReplicaSet from YAML | `kubectl create -f rs.yaml` |
| `kubectl get rs` | List all ReplicaSets | `kubectl get rs` |
| `kubectl get rs <name>` | Get details of a specific ReplicaSet | `kubectl get rs my-rs -o wide` |
| `kubectl describe rs <name>` | Show detailed ReplicaSet info | `kubectl describe rs my-rs` |
| `kubectl edit rs <name>` | Edit ReplicaSet configuration | `kubectl edit rs my-rs` |
| `kubectl delete rs <name>` | Delete a ReplicaSet | `kubectl delete rs my-rs` |
| `kubectl delete rs -l <label>` | Delete ReplicaSets by label | `kubectl delete rs -l app=web` |
| `kubectl scale rs <name> --replicas=<n>` | Scale ReplicaSet to desired replicas | `kubectl scale rs my-rs --replicas=5` |
| `kubectl label rs <name> key=value` | Add/modify label on a ReplicaSet | `kubectl label rs my-rs env=prod --overwrite` |
| `kubectl annotate rs <name> key=value` | Add annotation to a ReplicaSet | `kubectl annotate rs my-rs owner="team-a"` |
| `kubectl rollout status rs/<name>` | Check rollout status of ReplicaSet | `kubectl rollout status rs/my-rs` |
| `kubectl get pods -l <label>` | Get Pods managed by ReplicaSet | `kubectl get pods -l app=web` |
| `kubectl delete pod <pod>` | Delete Pod (ReplicaSet auto-recreates) | `kubectl delete pod my-rs-abc123` |
| `kubectl set image rs/<name> <container>=<image>` | Update container image in ReplicaSet | `kubectl set image rs/my-rs nginx=nginx:1.27.0` |
| `kubectl expose rs <name>` | Expose ReplicaSet as Service | `kubectl expose rs my-rs --port=80 --target-port=8080 --type=NodePort` |
| `kubectl patch rs <name>` | Patch ReplicaSet | `kubectl patch rs my-rs -p '{"spec":{"replicas":4}}'` |

---

## 📌 NOTES
- ReplicaSets usually aren’t created directly in production — **Deployments** are preferred.  
- If you delete a Pod managed by a ReplicaSet, it **auto-creates** a replacement Pod.  
- Useful mainly for **understanding how Deployments work internally**.  

---
