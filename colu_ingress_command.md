# 🌐 Kubernetes Imperative Commands – Ingress, PV, PVC & StorageClasses

---

## 🌐 INGRESS COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create ingress <name> --rule="<host>/<path>=<service>:<port>"` | Create a basic ingress rule | `kubectl create ingress app-ingress --rule="myapp.com/=myservice:80"` |
| `kubectl get ingress` | List all ingresses | `kubectl get ingress` |
| `kubectl describe ingress <name>` | Describe an ingress | `kubectl describe ingress app-ingress` |
| `kubectl get ingress <name> -o yaml` | View ingress in YAML | `kubectl get ingress app-ingress -o yaml` |
| `kubectl delete ingress <name>` | Delete an ingress | `kubectl delete ingress app-ingress` |
| `kubectl create ingress <name> --rule="foo.com/foo=foo:8080,tls=foo-tls"` | Create ingress with TLS | `kubectl create ingress secure-ing --rule="foo.com/=foo:8080,tls=foo-tls"` |
| `kubectl create ingress <name> --dry-run=client -o yaml > ingress.yaml` | Generate ingress YAML | `kubectl create ingress app-ingress --rule="myapp.com/=myservice:80" --dry-run=client -o yaml > ingress.yaml` |

---

## 📦 PERSISTENT VOLUME (PV) COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create -f pv.yaml` | Create a PersistentVolume from YAML | `kubectl create -f pv.yaml` |
| `kubectl get pv` | List all PVs | `kubectl get pv` |
| `kubectl describe pv <name>` | Describe a PV | `kubectl describe pv my-pv` |
| `kubectl get pv <name> -o yaml` | View PV YAML | `kubectl get pv my-pv -o yaml` |
| `kubectl delete pv <name>` | Delete a PV | `kubectl delete pv my-pv` |

---

## 📦 PERSISTENT VOLUME CLAIM (PVC) COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create -f pvc.yaml` | Create PVC from YAML | `kubectl create -f pvc.yaml` |
| `kubectl get pvc` | List all PVCs | `kubectl get pvc` |
| `kubectl describe pvc <name>` | Describe a PVC | `kubectl describe pvc my-pvc` |
| `kubectl get pvc <name> -o yaml` | View PVC YAML | `kubectl get pvc my-pvc -o yaml` |
| `kubectl delete pvc <name>` | Delete a PVC | `kubectl delete pvc my-pvc` |

---

## 🗄️ STORAGE CLASS COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl get sc` | List all storage classes | `kubectl get sc` |
| `kubectl describe sc <name>` | Describe a storage class | `kubectl describe sc standard` |
| `kubectl create -f sc.yaml` | Create a storage class from YAML | `kubectl create -f sc.yaml` |
| `kubectl delete sc <name>` | Delete a storage class | `kubectl delete sc fast-storage` |
| `kubectl get sc <name> -o yaml` | View SC YAML | `kubectl get sc standard -o yaml` |

---

## 📌 NOTES

- **Ingress** exposes HTTP/HTTPS routes from outside the cluster to services inside. Needs an ingress controller (like Nginx, Traefik).  
- **PersistentVolume (PV):** Cluster-level storage resource (NFS, EBS, hostPath, etc.).  
- **PersistentVolumeClaim (PVC):** A request for storage by a user/pod.  
- **StorageClass (SC):** Defines how PVs are provisioned (dynamic provisioning).  
- PVCs can automatically bind to PVs via StorageClasses.  

---
