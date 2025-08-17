# ❤️ Kubernetes Probes – Imperative Commands Cheat Sheet

Probes are used by Kubelet to check the **health of containers**:
- **Liveness Probe** → Is the container still running? Restart if failing.
- **Readiness Probe** → Is the app ready to receive traffic?
- **Startup Probe** → Did the app start successfully? (Useful for slow apps)

---

## 🚀 WORKFLOW (Imperative Style)

| **Step** | **Command** | **Description** |
|----------|-------------|-----------------|
| 1 | `kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml` | Generate base Pod YAML |
| 2 | `vim pod.yaml` (or editor) | Edit YAML to add **probes** |
| 3 | `kubectl apply -f pod.yaml` | Create Pod with probes |
| 4 | `kubectl describe pod mypod` | Check probe status |
| 5 | `kubectl logs mypod` | Debug container failing due to probes |

---

## 🔧 PROBE YAML SNIPPETS

👉 Add these under `containers:` → `livenessProbe`, `readinessProbe`, `startupProbe`.

### 1. **Liveness Probe** (HTTP)
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

### 1. **Readiness Probe** (HTTP)
```yaml
readinessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 5
  periodSeconds: 10
```

### 1. **StartUp Probe** (HTTP)
```yaml
startupProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 30
```


### 📌 NOTES
- InitialDelaySeconds → Wait before first check.
- PeriodSeconds → How often to run the check.
- FailureThreshold → Number of failures before container is considered unhealthy.
- SuccessThreshold → Number of successes required to mark healthy again.

