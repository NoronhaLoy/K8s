# ⚡ Kubernetes Imperative Commands – Complete Cheat Sheet

## 1. Pods
```bash
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx -l app=web,tier=frontend
kubectl run redis --image=redis --env="MODE=prod" --env="DEBUG=false"
kubectl run nginx --image=nginx --requests="cpu=200m,memory=128Mi" --limits="cpu=500m,memory=256Mi"
kubectl run nginx --image=nginx --port=80 --expose
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

## 2. Deployments
```bash
kubectl create deployment myapp --image=nginx
kubectl create deployment myapp --image=nginx --replicas=3
kubectl expose deployment myapp --type=NodePort --port=80
kubectl scale deployment myapp --replicas=10
kubectl set image deployment/myapp nginx=nginx:1.21
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
```

## 3. ReplicaSets
```bash
kubectl scale rs my-rs --replicas=4
```

## 4. DaemonSets
```bash
kubectl create daemonset fluentd --image=fluent/fluentd
```

## 5. StatefulSets
```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > statefulset.yaml
```

## 6. Jobs
```bash
kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'
kubectl create job batch-job --image=busybox -- /bin/sh -c "echo Job" --dry-run=client -o yaml > job.yaml
```

## 7. CronJobs
```bash
kubectl create cronjob hello --image=busybox --schedule="*/5 * * * *" -- echo "Hello World"
kubectl create cronjob backup --image=busybox --schedule="0 0 * * *" -- sh -c "echo Backup"
```

## 8. Services
```bash
kubectl expose pod nginx --port=80 --target-port=80 --type=ClusterIP
kubectl expose deployment myapp --type=NodePort --port=80 --target-port=8080
kubectl expose deployment myapp --type=LoadBalancer --port=80
```

## 9. Namespaces
```bash
kubectl create namespace dev
kubectl create namespace staging --dry-run=client -o yaml > ns.yaml
```

## 10. ConfigMaps
```bash
kubectl create configmap app-config --from-literal=ENV=prod --from-literal=DEBUG=true
kubectl create configmap app-config --from-env-file=config.env
kubectl create configmap app-config --from-file=app.properties
kubectl create configmap app-config --from-file=config-dir/
```

## 11. Secrets
```bash
kubectl create secret generic db-secret --from-literal=USER=admin --from-literal=PASSWORD=pass123
kubectl create secret generic app-secret --from-file=creds.txt
kubectl create secret tls tls-secret --cert=cert.crt --key=cert.key
```

## 12. Ingress
```bash
kubectl create ingress my-ing --rule="app.example.com/=myapp:80"
```

## 13. ServiceAccounts / RBAC
```bash
kubectl create serviceaccount my-sa
kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:my-sa
kubectl create clusterrole pod-reader --verb=get --verb=list --resource=pods
kubectl create clusterrolebinding read-pods-global --clusterrole=pod-reader --serviceaccount=default:my-sa
```

## 14. Storage
```bash
kubectl create pvc mypvc --storage=1Gi --access-modes=ReadWriteOnce
kubectl create pv mypv --capacity=1Gi --access-modes=ReadWriteOnce --hostpath=/data
```

## 15. Resource Management
```bash
kubectl autoscale deployment myapp --min=2 --max=10 --cpu-percent=80
kubectl set resources deployment myapp --limits=cpu=500m,memory=256Mi --requests=cpu=200m,memory=128Mi
```

## 16. Node Management
```bash
kubectl cordon node01
kubectl drain node01 --ignore-daemonsets
kubectl uncordon node01
```

## 17. Debugging
```bash
kubectl logs mypod
kubectl logs -f mypod
kubectl exec -it mypod -- /bin/sh
kubectl run debug --image=busybox --rm -it -- sh
```

## 18. Editing / Patching
```bash
kubectl edit deployment myapp
kubectl patch deployment myapp -p '{"spec":{"replicas":5}}'
```

## 19. Labeling / Annotating
```bash
kubectl label pod mypod env=prod
kubectl label pod mypod env=dev --overwrite
kubectl annotate pod mypod description="Test pod"
```

## 20. Taints & Tolerations
```bash
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 key:NoSchedule-
```

## 21. Rollouts
```bash
kubectl rollout pause deployment/myapp
kubectl rollout resume deployment/myapp
```

## 22. Generate YAML (Dry Run)
```bash
kubectl create deployment myapp --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl expose deployment myapp --port=80 --dry-run=client -o yaml > svc.yaml
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```
