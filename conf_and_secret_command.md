# 🔑 Kubernetes Imperative Commands – ConfigMaps & Secrets

## 📋 CONFIGMAP COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create configmap <name> --from-literal=<key>=<value>` | Create ConfigMap from literal values | `kubectl create configmap app-config --from-literal=ENV=prod` |
| `kubectl create configmap <name> --from-file=<path>` | Create ConfigMap from file | `kubectl create configmap app-config --from-file=app.properties` |
| `kubectl create configmap <name> --from-file=<key>=<path>` | Create ConfigMap with custom key from file | `kubectl create configmap app-config --from-file=db=./db.conf` |
| `kubectl create configmap <name> --from-env-file=<path>` | Create ConfigMap from env file | `kubectl create configmap app-config --from-env-file=app.env` |
| `kubectl get configmaps` | List all ConfigMaps | `kubectl get configmaps` |
| `kubectl describe configmap <name>` | Describe a ConfigMap | `kubectl describe configmap app-config` |
| `kubectl get configmap <name> -o yaml` | View ConfigMap in YAML | `kubectl get configmap app-config -o yaml` |
| `kubectl delete configmap <name>` | Delete a ConfigMap | `kubectl delete configmap app-config` |
| `kubectl create configmap <name> --dry-run=client -o yaml > cm.yaml` | Generate YAML for ConfigMap | `kubectl create configmap app-config --from-literal=ENV=dev --dry-run=client -o yaml > cm.yaml` |

---

## 📋 SECRET COMMANDS

| **Command** | **Description** | **Example** |
|-------------|-----------------|-------------|
| `kubectl create secret generic <name> --from-literal=<key>=<value>` | Create generic Secret from literal | `kubectl create secret generic db-secret --from-literal=USER=admin --from-literal=PWD=12345` |
| `kubectl create secret generic <name> --from-file=<path>` | Create Secret from file | `kubectl create secret generic db-secret --from-file=./username.txt` |
| `kubectl create secret generic <name> --from-file=<key>=<path>` | Create Secret with custom key from file | `kubectl create secret generic db-secret --from-file=user=./username.txt` |
| `kubectl create secret generic <name> --from-env-file=<path>` | Create Secret from env file | `kubectl create secret generic db-secret --from-env-file=secret.env` |
| `kubectl create secret docker-registry <name> --docker-server=<url> --docker-username=<user> --docker-password=<pass> --docker-email=<email>` | Create Docker registry Secret | `kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=myuser --docker-password=mypwd --docker-email=my@email.com` |
| `kubectl get secrets` | List all Secrets | `kubectl get secrets` |
| `kubectl describe secret <name>` | Describe a Secret | `kubectl describe secret db-secret` |
| `kubectl get secret <name> -o yaml` | View Secret YAML (base64 encoded) | `kubectl get secret db-secret -o yaml` |
| `kubectl get secret <name> -o jsonpath="{.data.<key>}" | base64 --decode` | Decode secret value | `kubectl get secret db-secret -o jsonpath="{.data.USER}" | base64 --decode` |
| `kubectl delete secret <name>` | Delete a Secret | `kubectl delete secret db-secret` |
| `kubectl create secret generic <name> --dry-run=client -o yaml > secret.yaml` | Generate YAML for Secret | `kubectl create secret generic db-secret --from-literal=USER=admin --dry-run=client -o yaml > secret.yaml` |

---

## 📌 NOTES

- **ConfigMaps** store **non-sensitive** configuration (env variables, config files).  
- **Secrets** store **sensitive** data (passwords, API keys, certificates).  
- Secrets values are **base64-encoded**, not encrypted (use encryption at rest for extra security).  
- Both can be mounted as **environment variables** or as **volumes** into Pods.  
- Use **docker-registry secret** when pulling images from private registries.  

---
