# Runbook: Мессенджер в Kubernetes

## Различия dev vs prod

| Параметр | dev | prod |
|---|---|---|
| Реплики (frontend/bff/user-service/message-service) | 1 | 2 |
| Resources (CPU/Memory) | базовые (150m/192Mi) | повышенные (300m/384Mi) |
| Теги образов | `latest` | `1.0.0` (фиксированный semver) |
| nodeAffinity | не задана | обязательна (patches/affinity.yaml) |
| Метка `env` | `dev` | `prod` |

### nodeAffinity в prod (docs/04-node-affinity-task.md)

| Сервис | Правило |
|---|---|
| postgres, minio | `requiredDuringScheduling`: `workload=system` |
| frontend, bff, user-service | `requiredDuringScheduling`: `workload=app` |
| message-service | `required` `workload=app` + `preferred` (weight 100) `disk=fast` |

---

## Как запустить (dev, Minikube)

```bash
# Применить все манифесты одной командой
kubectl apply -k k8s/overlays/dev/

# Проверить готовность
kubectl get pods -n messenger
```

## Как установить Argo CD и применить Application

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=120s
kubectl apply -f argocd/application.yaml
kubectl get applications -n argocd
```

---

## Подтверждение проверок (результаты выполнения)

### 1. `kustomize build k8s/overlays/dev` — без ошибок 

```
$ kubectl kustomize k8s/overlays/dev | grep "^kind:" | sort | uniq -c
      2 kind: ConfigMap
      6 kind: Deployment
      2 kind: Job
      1 kind: Namespace
      2 kind: PersistentVolumeClaim
      3 kind: Secret
      6 kind: Service
      1 kind: StorageClass
```

### 2. `kustomize build k8s/overlays/prod` — без ошибок 

```
$ kubectl kustomize k8s/overlays/prod | grep "^kind:" | sort | uniq -c
      2 kind: ConfigMap
      6 kind: Deployment
      2 kind: Job
      1 kind: Namespace
      2 kind: PersistentVolumeClaim
      3 kind: Secret
      6 kind: Service
      1 kind: StorageClass
```

### 3. Все поды в состоянии Running/Completed 

```
$ kubectl get pods -n messenger -o wide
NAME                               READY   STATUS      RESTARTS   AGE   IP            NODE
bff-74cc6bc7c7-d478l               1/1     Running     0          55m   10.244.0.60   minikube
frontend-844bd4c45d-76rkr          1/1     Running     0          55m   10.244.0.65   minikube
message-service-7549fb5fd5-cr2ts   1/1     Running     0          21m   10.244.0.75   minikube
migrate-messages-gzn68             0/1     Completed   0          55m   10.244.0.63   minikube
migrate-users-ms6v9                0/1     Completed   0          55m   10.244.0.64   minikube
minio-75f6c95c7b-b4pj4             1/1     Running     0          55m   10.244.0.61   minikube
postgres-66f696f4ff-mm6r7          1/1     Running     0          55m   10.244.0.62   minikube
user-service-9c7896794-vmhhh       1/1     Running     2          55m   10.244.0.66   minikube
```

### 4. Frontend доступен снаружи кластера 

```
$ kubectl get svc frontend -n messenger
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
frontend   NodePort   10.104.86.82   <none>        80:30080/TCP   55m

URL: http://192.168.49.2:30080
```

### 5. Миграции выполнены 

```
$ kubectl get jobs -n messenger
NAME               STATUS     COMPLETIONS   DURATION   AGE
migrate-messages   Complete   1/1           25s        55m
migrate-users      Complete   1/1           23s        55m
```

### 6. Загрузка файлов через S3 CSI работает 

```
$ kubectl exec -n messenger deployment/message-service -- touch /app/uploads/verify_checklist.txt
$ kubectl exec -n messenger deployment/message-service -- ls -la /app/uploads/
drwxrwxrwx  1 root  root     0 Jan  1  1970 .
drwxr-xr-x  1 root  root  4096 Apr 26 11:34 ..
-rw-r--r--  1 root  root  5972 May 16 19:30 139f1056-e3fb-446d-9161-4ca5d9aef120.webm
-rw-r--r--  1 root  root  500532 May 16 19:28 63d09d12-7c2c-400d-a542-8559004c01ca.jpg
-rw-r--r--  1 root  root  77744 May 16 19:30 b3299377-68eb-44d3-b8b4-8daec7a23e5d.webm
-rw-r--r--  1 root  root  67086 May 16 19:30 ce482d2f-bf15-4549-8846-1d5c557babf2.webm
-rw-r--r--  1 root  root     0 May 16 19:49 verify_checklist.txt
```

### 7. Объекты появляются в S3 bucket (MinIO) 

```
$ mc ls --recursive local/pvc-7abf6282-a086-4dbc-a9f5-00338cdde2df/csi-fs/
[2026-05-16 19:30:02]  5.8KiB 139f1056-e3fb-446d-9161-4ca5d9aef120.webm
[2026-05-16 19:28:43]  489KiB 63d09d12-7c2c-400d-a542-8559004c01ca.jpg
[2026-05-16 19:30:19]   76KiB b3299377-68eb-44d3-b8b4-8daec7a23e5d.webm
[2026-05-16 19:30:07]   66KiB ce482d2f-bf15-4549-8846-1d5c557babf2.webm
[2026-05-16 19:49:10]     0B  verify_checklist.txt
```

### 8. nodeAffinity реализована в prod overlay ✅

Патч `k8s/overlays/prod/patches/affinity.yaml` задаёт:
- `postgres`, `minio` → `required: workload=system`
- `frontend`, `bff`, `user-service` → `required: workload=app`
- `message-service` → `required: workload=app` + `preferred (weight=100): disk=fast`

```bash
# Установить метки перед деплоем prod:
kubectl label node minikube workload=system disk=fast

# Проверить Affinity у пода:
kubectl describe pod -n messenger -l app=message-service | grep -A15 "Affinity"
```

### 9. S3 CSI StorageClass (ch.ctrox.csi.s3-driver, mounter: s3fs) ✅

```
$ kubectl get storageclass csi-s3
NAME     PROVISIONER              RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION
csi-s3   ch.ctrox.csi.s3-driver   Delete          Immediate           false
```
