# k8s-messenger-gitops

Kubernetes-конфигурация для микросервисного мессенджера.  
Развёртывание через Kustomize (`dev` / `prod`) с GitOps-деплоем через Argo CD.

## Стек

| Компонент | Образ |
|---|---|
| frontend | `mablinov2704/frontend:latest` |
| bff | `mablinov2704/bff:latest` |
| user-service | `mablinov2704/user-service:latest` |
| message-service | `mablinov2704/message-service:latest` |
| postgres | `postgres:16-alpine` |
| minio (S3) | `minio/minio:latest` |

Файлы в `message-service` монтируются через **S3 CSI** (`ch.ctrox.csi.s3-driver`, маунтер `s3fs`).

## Структура репозитория

```
k8s/
  base/          — общие манифесты (namespace, configmap, postgres, сервисы, S3 StorageClass)
  overlays/
    dev/         — dev: 1 реплика, базовые ресурсы, latest-образы
    prod/        — prod: 2 реплики, повышенные ресурсы, nodeAffinity, фиксированные теги
argocd/
  application.yaml   — Application для messager-dev и messager-prod
docs/
  runbook.md     — Подробный runbook + различия dev/prod + команды проверки
```

## Быстрый запуск (dev, Minikube)

```bash
# 1. Применить все манифесты одной командой
kubectl apply -k k8s/overlays/dev/

# 2. Дождаться готовности
kubectl wait --for=condition=Ready pod -l app=frontend -n messenger --timeout=120s
kubectl get pods -n messenger

# 3. Открыть frontend
minikube service frontend -n messenger
# или: http://$(minikube ip):30080
```

## Проверка работоспособности

```bash
# Все поды Running/Completed
kubectl get pods -n messenger

# S3 CSI — запись файлов работает
kubectl exec -n messenger deployment/message-service -- touch /app/uploads/test.txt
kubectl exec -n messenger deployment/message-service -- ls /app/uploads/

# Оба overlay собираются без ошибок
kubectl kustomize k8s/overlays/dev
kubectl kustomize k8s/overlays/prod
```

## GitOps через Argo CD

```bash
# Установка Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Применить Application
kubectl apply -f argocd/application.yaml

# Проверить статус (Synced/Healthy)
kubectl get applications -n argocd
```

Подробнее — в [`docs/runbook.md`](docs/runbook.md).
