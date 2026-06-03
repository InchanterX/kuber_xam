# Структура

**argocd/**
- `application-dev.yaml` — ArgoCD Application для dev
- `application-prod.yaml` — ArgoCD Application для prod

**k8s/base/** — базовые манифесты
- Frontend Deployment + Service
- BFF Deployment + Service
- User Service Deployment + Service
- Message Service Deployment + Service
- PostgreSQL Deployment + Service + PVC + Init ConfigMap
- MinIO Deployment + Service + Secret
- S3 PersistentVolume + PersistentVolumeClaim
- CSI-S3 Secret (для монтирования S3 в поды)
- Ingress (маршрутизация)
- DNS DaemonSet (корректировка DNS)
- Jobs для миграции БД (migrate-users, migrate-messages)
- ConfigMap + Secret для конфигурации

**k8s/overlays/dev** — изменения для разработки
- Уменьшенное количество реплик
- Меньше ресурсов (CPU/Memory)
- Dev домены для Ingress

**k8s/overlays/prod** — изменения для production
- Больше реплик для надёжности
- Pod Affinity/Anti-Affinity правила
- Оптимизированные ресурсы
- Prod домены и SSL

# Быстрый старт

## Dev
```bash
kubectl apply -k k8s/overlays/dev
```

## Prod с ArgoCD
```bash
kubectl apply -f argocd/application-prod.yaml
```

# Основные файлы

| Файл | Назначение |
|------|-----------|
| `secret.yaml` | БД пароли, API ключи (шифровать в prod) |
| `configmap.yaml` | Конфиг параметры (LOG_LEVEL, SERVICE_DISCOVERY) |
| `kustomization.yaml` | Список ресурсов для Kustomize |
| `*-deployment.yaml` | Deployment с containers, resources, probes |
| `*-service.yaml` | Service для внутренней/внешней связи |
| `*-job.yaml` | Job для миграций БД перед запуском |
| `ingress.yaml` | HTTP маршруты с доменами |

# Как это работает

1. **Base** содержит все манифесты без окружение-специфичных параметров
2. **Overlays** применяют Strategic Merge Patches для изменения:
   - Количества реплик (replicas.yaml)
   - CPU/Memory запросов и лимитов (resources.yaml)
   - Доменов Ingress (ingress.yaml)
   - Pod размещения (affinity.yaml для prod)
3. ArgoCD синхронизирует конфиги из репозитория в кластер

# Перед запуском

1. Установить значения в `k8s/base/secret.yaml`:
   ```yaml
   POSTGRES_USER: <user>
   POSTGRES_PASSWORD: <password>
   DATABASE_URL: postgresql://<user>:<pass>@postgres:5432/app_db
   MINIO_ROOT_USER: <user>
   MINIO_ROOT_PASSWORD: <password>
   JWT_SECRET: <32+ символа>
   ```

2. Обновить домены в `k8s/overlays/*/patches/ingress.yaml`


# Проверка развёртывания

```bash
kubectl get pods
kubectl get svc
kubectl get pvc
kubectl logs job/migrate-users
```

# Структура с точки зрения паттерна Kustomize

```
base/
  ├─ resources              (все ресурсы)
  └─ kustomization.yaml     (список ресурсов + общие настройки)

overlays/
  ├─ dev/
  │  ├─ kustomization.yaml  (bases: ../../base + patches)
  │  └─ patches/            (стратегические изменения для dev)
  └─ prod/
     ├─ kustomization.yaml
     └─ patches/            (стратегические изменения для prod)
```

При выполнении `kubectl apply -k overlays/dev`:
1. Читаются ресурсы из `base`
2. Применяются патчи из `dev/patches`
3. Результат отправляется в кластер
