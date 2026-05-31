# Runbook: развёртывание и проверка

## Предпосылки

- Кластер с ≥2 классами узлов (метки `workload=system` / `workload=app`, часть `app` с `disk=fast`).
- Установленный CSI-S3 драйвер (`ru.yandex.s3.csi`) — см. `csi-s3/`.
- `kubectl` (встроенный `kubectl kustomize`).

## 0. Узлы (один раз на кластер)

```bash
kubectl label node <system-node> workload=system --overwrite
kubectl label node <app-node-1>  workload=app    --overwrite
kubectl label node <app-node-2>  workload=app    --overwrite
kubectl label node <fast-node>   workload=app disk=fast --overwrite
```

## 1. CSI-S3 драйвер (один раз на кластер)

```bash
kubectl apply -f csi-s3/driver.yaml -f csi-s3/provisioner.yaml \
              -f csi-s3/csi-s3.yaml -f csi-s3/storageclass.yaml
kubectl -n kube-system rollout status ds/csi-s3
```

> Драйвер `ru.yandex.s3.csi` (geesefs) — поддерживаемый преемник `ch.ctrox.csi.s3-driver`
> из методички; работает на актуальных версиях Kubernetes (проверено на k3s v1.33).

## 2. Деплой приложения

Напрямую через kustomize:

```bash
kubectl apply -k k8s/overlays/dev      # namespace messager
# или prod:
kubectl apply -k k8s/overlays/prod     # namespace messager-prod
```

Через GitOps (Argo CD) — см. `docs` ниже и `argocd/`.

## 3. Проверки (соответствуют чек-листу методички)

```bash
# 1. Сборка overlay без ошибок
kubectl kustomize k8s/overlays/dev  >/dev/null && echo OK
kubectl kustomize k8s/overlays/prod >/dev/null && echo OK

# 2. Все pod-ы Running/Completed
kubectl get pods -n messager -o wide

# 3+4. Доступность frontend и цепочка frontend -> bff -> services
kubectl port-forward -n messager svc/frontend 18080:80 &
curl -s -X POST localhost:18080/api/v1/users -d '{"name":"alice"}' -H 'Content-Type: application/json'
curl -s -X POST localhost:18080/api/v1/users -d '{"name":"bob"}'   -H 'Content-Type: application/json'
curl -s -X POST localhost:18080/api/v1/messages \
     -H 'Content-Type: application/json' \
     -d '{"sender_id":"<alice-id>","receiver_id":"<bob-id>","text":"hi"}'

# 5. Загрузка файла -> объект в S3 bucket
curl -s -X POST localhost:18080/api/v1/files -F 'file=@/path/img.png;type=image/png'
kubectl exec -n messager de:message-service -- ls -la /app/uploads   # geesefs mount == bucket

# 6. Placement по affinity
kubectl get pods -n messager -o wide
kubectl describe pod -n messager -l app=message-service | sed -n '/Node-Selectors/,/Events/p'

# 7. Argo CD
kubectl get applications -n argocd
```

## Результаты проверки этого развёртывания (dev)

- Все pod-ы `Running/Completed`; миграции (`migrate-users`, `migrate-messages`) — `Completed`.
- Цепочка `frontend → bff → user-service/message-service → postgres` — рабочая
  (регистрация пользователей, отправка/чтение сообщений).
- Загрузка файла: объект `<uuid>.png` появляется в bucket `message-uploads` (geesefs),
  доступен через `GET /api/v1/files/<id>` и **сохраняется после рестарта pod** message-service.
- Placement:
  - `postgres`, `minio` → узел `workload=system`;
  - `frontend`, `bff`, `user-service` → узлы `workload=app`;
  - `message-service` → узел `workload=app` **с `disk=fast`** (сработал soft-preference).

## Типичные проблемы

- **`MountVolume.MountDevice ... DeadlineExceeded` / `lookup minio: no such host`** —
  endpoint в `csi-s3-secret` должен быть FQDN (`minio.<ns>.svc.cluster.local:9000`),
  т.к. geesefs монтирует внутри pod-а драйвера в namespace `kube-system`.
- **Job миграций падает** — проверьте, что `postgres` поднялся и init-скрипт создал
  БД `messager_users` / `messager_messages` (initContainer `wait-for-postgres`).
- **PVC не биндится** — статический PV из `overlays/*/s3-pv.yaml` имеет `claimRef` на нужный namespace.
