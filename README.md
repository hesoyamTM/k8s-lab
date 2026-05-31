# Messager в Kubernetes (kustomize + S3 CSI + nodeAffinity + Argo CD)

Лабораторная по развёртыванию микросервисного мессенджера в Kubernetes по практикам GitOps.
Исходное приложение и методичка: <https://github.com/tenzm/lab-k8s-messager>.

## Состав приложения

`frontend` (nginx+SPA) → `bff` → {`user-service`, `message-service`} → `postgres`
(две БД: `messager_users`, `messager_messages`). Файлы сообщений `message-service`
хранятся в S3 (MinIO), смонтированном **через CSI** в `/app/uploads`.

## Структура репозитория

```
k8s/
  base/                 # общие манифесты (Deployments, Services, ConfigMap/Secret,
                        # миграции-Jobs, MinIO, PVC, Ingress, nodeAffinity)
  overlays/
    dev/                # namespace messager, реплики 1, образы :latest
    prod/               # namespace messager-prod, реплики >=2, образы по digest
argocd/                 # Argo CD Application для dev и prod (GitOps-автосинк)
csi-s3/                 # установка CSI-S3 драйвера (ru.yandex.s3.csi / geesefs)
docs/                   # runbook и таблица отличий dev/prod
```

## Быстрый старт

```bash
# 1. метки узлов и CSI-драйвер — см. docs/runbook.md (разделы 0 и 1)
# 2. деплой
kubectl apply -k k8s/overlays/dev
# 3. проверка
kubectl get pods -n messager -o wide
```

## Ключевые требования и где они реализованы

| Требование | Где |
|---|---|
| Все сервисы + миграции + postgres | `k8s/base/*` |
| Файлы только через S3 **CSI-mount** | `k8s/base/s3-pvc.yaml`, `k8s/overlays/*/s3-pv.yaml`, `csi-s3/` |
| nodeAffinity (system/app/fast) | `affinity:` в каждом Deployment в `k8s/base/*` |
| kustomize base + overlays dev/prod | `k8s/base`, `k8s/overlays/{dev,prod}` |
| Argo CD автосинк (`automated`+`prune`+`selfHeal`) | `argocd/application-*.yaml` |

Подробности проверки — в [docs/runbook.md](docs/runbook.md);
отличия окружений — в [docs/dev-prod-diff.md](docs/dev-prod-diff.md).

## GitOps (Argo CD)

```bash
kubectl create ns argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# подставить repoURL/branch в argocd/application-*.yaml, затем:
kubectl apply -f argocd/application-dev.yaml
```

Argo CD отслеживает `k8s/overlays/dev` в этом репозитории и автоматически применяет
изменения (`prune` удаляет лишнее, `selfHeal` возвращает desired state при ручном drift).
