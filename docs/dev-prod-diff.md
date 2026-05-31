# Отличия overlay `dev` и `prod`

Оба overlay собираются из общего `k8s/base` и отличаются только наложениями.

| Параметр | dev (`k8s/overlays/dev`) | prod (`k8s/overlays/prod`) |
|---|---|---|
| Namespace | `messager` | `messager-prod` |
| Реплики `frontend`/`bff`/`user-service`/`message-service` | 1 | 2 |
| Реплики `postgres`/`minio` | 1 | 1 (stateful, одиночные) |
| Образы | `mablinov2704/*:latest` (плавающий тег) | `mablinov2704/*@sha256:...` (фиксированный digest — воспроизводимость) |
| Ресурсы `frontend` | 100m/128Mi → 200m/256Mi | 200m/256Mi → 400m/512Mi |
| Ресурсы backend-сервисов | 150m/192Mi → 300m/384Mi | 300m/384Mi → 600m/768Mi |
| Ingress host | `dev.messager.local` | `messager.example.com` |
| S3 PV | `dev-s3-pv-uploads`, бакет `message-uploads` | `prod-s3-pv-uploads`, бакет `message-uploads/prod` |
| CSI endpoint (FQDN) | `minio.messager.svc.cluster.local:9000` | `minio.messager-prod.svc.cluster.local:9000` |
| Метка окружения | `app.kubernetes.io/environment: dev` | `app.kubernetes.io/environment: prod` |

## Что общего (в `base`, не дублируется в overlay)

- Все Deployments/Services/ConfigMap/Secret/Job/PVC.
- **nodeAffinity** (одинаковые правила в обоих окружениях):
  - `postgres`, `minio` → hard `workload=system`;
  - `frontend`, `bff`, `user-service` → hard `workload=app`;
  - `message-service` → hard `workload=app` + soft `disk=fast`.
- Probes (readiness/liveness), миграции БД (goose Jobs), создание bucket.

## Механики kustomize

- `namespace:` — переопределение namespace в overlay.
- `replicas:` — изменение числа реплик в prod.
- `images:` — `newTag: latest` (dev) и `digest:` (prod).
- `patches:` — повышение ресурсов, prod-host Ingress, prod-endpoint CSI-секрета.
- `labels: includeSelectors: false` — метки окружения без затрагивания иммутабельных селекторов.
