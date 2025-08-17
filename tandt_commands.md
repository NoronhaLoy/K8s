# ☢️ Kubernetes Imperative Commands – Taints & Tolerations

**Taints**: Applied to nodes to repel Pods.  
**Tolerations**: Applied to Pods to allow scheduling on tainted nodes.

---

## 📋 TAINT COMMANDS (Applied on Nodes)

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl taint nodes <node> <key>=<value>:<effect>` | Add taint to a node | `kubectl taint nodes node1 env=prod:NoSchedule` |
| `kubectl taint nodes <node> <key>-` | Remove taint from a node | `kubectl taint nodes node1 env-` |
| `kubectl taint nodes <node> <key>=<value>:<effect>-` | Remove specific taint with key/value/effect | `kubectl taint nodes node1 env=prod:NoSchedule-` |
| `kubectl describe node <node>` | View taints on a node | `kubectl describe node node1` |
| `kubectl get nodes -o json | jq '.items[].spec.taints'` | View all taints across cluster | `kubectl get nodes -o json | jq '.items[].spec.taints'` |

---

## 📋 TOLERATION COMMANDS (Applied on Pods)

Tolerations are specified inside Pod spec (`spec.tolerations`).  
Imperatively, we usually generate YAML with `--dry-run`.

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl run <pod> --image=<image> --dry-run=client -o yaml > pod.yaml` | Generate Pod manifest | `kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > pod.yaml` |
| *(then edit `pod.yaml` to add tolerations)* | Add tolerations under `spec.tolerations` | ```yaml\nspec:\n  tolerations:\n  - key: "env"\n    operator: "Equal"\n    value: "prod"\n    effect: "NoSchedule"\n``` |
| `kubectl create -f pod.yaml` | Create Pod with tolerations | `kubectl create -f pod.yaml` |
| `kubectl edit pod <name>` | Add toleration interactively | `kubectl edit pod nginx` |

---

## 📌 EFFECTS TYPES
- **NoSchedule** → New Pods won’t schedule unless tolerated.  
- **PreferNoSchedule** → Tries to avoid scheduling, but not guaranteed.  
- **NoExecute** → Evicts existing Pods unless tolerated.  

---

## 📌 NOTES
- **Taints = repelling rule** (on nodes).  
- **Tolerations = exemption rule** (on pods).  
- Pods without tolerations won’t run on tainted nodes.  
- Use `describe node` often to debug taints.  

---
