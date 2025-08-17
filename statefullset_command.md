 # 🗂️ Kubernetes Imperative Commands – StatefulSets

A **StatefulSet** is used to manage stateful applications (e.g., databases).  
It ensures Pods have **stable network IDs**, **persistent storage**, and **ordered deployment & scaling**.

---

## 📋 STATEFULSET COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create statefulset <name> --image=<image>` | Create a StatefulSet (basic) | `kubectl create statefulset web --image=nginx` |
| `kubectl create statefulset <name> --image=<image> --replicas=<n>` | Create StatefulSet with replicas | `kubectl create statefulset mysql --image=mysql --replicas=3` |
| `kubectl expose statefulset <name> --port=<port> --name=<svc>` | Expose StatefulSet as Service | `kubectl expose statefulset mysql --port=3306 --name=mysql-svc` |
| `kubectl get statefulsets` | List StatefulSets | `kubectl get statefulsets` |
| `kubectl get statefulset <name>` | Get details of a StatefulSet | `kubectl get statefulset mysql` |
| `kubectl describe statefulset <name>` | Describe StatefulSet in detail | `kubectl describe statefulset mysql` |
| `kubectl delete statefulset <name>` | Delete a StatefulSet | `kubectl delete statefulset mysql` |
| `kubectl scale statefulset <name> --replicas=<n>` | Scale StatefulSet Pods | `kubectl scale statefulset mysql --replicas=5` |
| `kubectl rollout status statefulset/<name>` | Check rollout status | `kubectl rollout status statefulset/mysql` |
| `kubectl rollout history statefulset/<name>` | View rollout history | `kubectl rollout history statefulset/mysql` |
| `kubectl rollout undo statefulset/<name>` | Rollback to previous version | `kubectl rollout undo statefulset/mysql` |
| `kubectl edit statefulset <name>` | Edit StatefulSet spec | `kubectl edit statefulset mysql` |
| `kubectl patch statefulset <name> -p '<json-patch>'` | Patch StatefulSet | `kubectl patch statefulset mysql -p '{"spec":{"replicas":4}}'` |
| `kubectl get pods -l app=<label>` | Get Pods of StatefulSet | `kubectl get pods -l app=mysql` |

---

## 📋 GENERATING YAML

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create statefulset <name> --image=<image> --dry-run=client -o yaml > sts.yaml` | Generate YAML for StatefulSet | `kubectl create statefulset web --image=nginx --dry-run=client -o yaml > web-sts.yaml` |
| *(edit `sts.yaml`)* | Add **volumeClaimTemplates**, serviceName, etc. | ```yaml\nvolumeClaimTemplates:\n- metadata:\n    name: data\n  spec:\n    accessModes: [ "ReadWriteOnce" ]\n    resources:\n      requests:\n        storage: 1Gi\n``` |
| `kubectl apply -f sts.yaml` | Apply YAML definition | `kubectl apply -f sts.yaml` |

---

## 📌 NOTES
- StatefulSets need a **Headless Service** (`clusterIP: None`) for stable DNS.  
- Pods in StatefulSet are named sequentially (`<name>-0`, `<name>-1`, ...).  
- Each Pod gets its **own PVC** via `volumeClaimTemplates`.  
- Use **StatefulSets for DBs, Kafka, Zookeeper, etc.**, not Deployments.  

---
