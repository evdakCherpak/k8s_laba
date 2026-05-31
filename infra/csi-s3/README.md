# Установка S3 CSI-драйвера (Helm)

Драйвер `yandex-cloud/k8s-csi-s3` (на базе GeeseFS) умеет монтировать S3-бакеты как
обычные папки внутри Pod'ов. Это **инфраструктура кластера** — ставится один раз вручную
(Argo CD её не разворачивает).

> Secret с ключами и `StorageClass` мы создаём **сами** в составе приложения
> (`k8s/base/s3-csi/` + патчи overlay'ев), поэтому в чарте отключаем их создание
> (см. [values.yaml](values.yaml)).

## Установка

```bash
# 1) подключаем репозиторий чартов
helm repo add yandex-s3 https://yandex-cloud.github.io/k8s-csi-s3/charts
helm repo update

# 2) ставим драйвер в namespace kube-system
helm install csi-s3 yandex-s3/csi-s3 \
  --namespace kube-system \
  -f infra/csi-s3/values.yaml
```

## Проверка

```bash
# Pod'ы драйвера должны быть Running (csi-s3 controller + DaemonSet на каждом узле)
kubectl -n kube-system get pods -l app=csi-s3

# зарегистрирован CSI-драйвер
kubectl get csidrivers | grep s3
```

## Удаление (если нужно переставить)

```bash
helm uninstall csi-s3 -n kube-system
```

## Как это связано с приложением

- Драйвер (этот Helm-релиз) = «движок», который умеет монтировать S3.
- `StorageClass csi-s3` (в нашем kustomize) = «рецепт»: какой бакет, какой эндпоинт MinIO,
  где взять ключи.
- `PVC uploads` (в нашем kustomize) = «заявка на диск» по этому рецепту.
- `message-service` монтирует `PVC uploads` в `/app/uploads` → файлы уходят в MinIO.
