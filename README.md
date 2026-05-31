# Мессенджер в Kubernetes (k3d) — kustomize + S3 CSI + Argo CD

Лабораторная: развернуть микросервисный мессенджер в Kubernetes, подключить файловое
хранилище через **S3 CSI**, настроить **nodeAffinity**, собрать **kustomize**
(base + overlays dev/prod) и организовать **GitOps через Argo CD**.

Кластер поднимается локально на **k3d** (k3s в Docker) на Ubuntu.

---

## 1. Архитектура

Приложение состоит из нескольких микросервисов. Кто кого вызывает:

```
браузер ─► Ingress (Traefik) ─► frontend (nginx:80)
                                    │
                                    ▼
                                  bff (8080)
                                  │      │
                                  ▼      ▼
                       user-service   message-service
                          (8081)         (8082)
                             │              │   │
                             ▼              ▼   ▼
                      postgres/usersdb   postgres/  MinIO (S3)
                                         messagesdb  через S3 CSI
```

| Компонент | Образ | Порт | Узел (nodeAffinity) |
|---|---|---|---|
| frontend | `mablinov2704/frontend` | 80 | `workload=app` |
| bff | `mablinov2704/bff` | 8080 | `workload=app` |
| user-service | `mablinov2704/user-service` | 8081 | `workload=app` |
| message-service | `mablinov2704/message-service` | 8082 | `workload=app` (hard) + `disk=fast` (soft) |
| postgres | `postgres:16-alpine` | 5432 | `workload=system` |
| minio | `minio/minio` | 9000/9001 | `workload=system` |
| миграции | `ghcr.io/kukymbr/goose-docker` | — | `workload=app` (Job) |

---

## 2. Структура репозитория

```
k8s/
├── base/                       # общий «рецепт» приложения (kustomize base)
│   ├── config/                 # ConfigMap + Secrets (порты, URL, DSN, креды)
│   ├── postgres/               # initdb(2 БД) + StatefulSet + Service
│   ├── minio/                  # Deployment + Service + Job создания бакета
│   ├── s3-csi/                 # StorageClass (GeeseFS) + PVC uploads (RWX)
│   ├── migrations/             # 2 Job: wait-db → cp /app/migrations → goose
│   ├── user-service/ message-service/ bff/ frontend/
│   └── kustomization.yaml
└── overlays/
    ├── dev/                    # ns messenger-dev, 1 реплика, latest, мягкая affinity
    └── prod/                   # ns messenger-prod, 2 реплики, образы по digest

argocd/
├── application-dev.yaml        # Argo CD Application → overlays/dev
└── application-prod.yaml       # Argo CD Application → overlays/prod

infra/                          # bootstrap (ставится ВРУЧНУЮ, один раз)
├── csi-s3/                     # установка S3-CSI драйвера через Helm
└── argocd/                     # установка самого Argo CD

docs/
└── report.md                   # пояснения, проверка, troubleshooting, отчёт
```

---

## 3. Предварительные требования

Установлены: `docker`, `k3d` (v5+), `kubectl`, `helm`. Docker-демон доступен текущему
пользователю. Аккаунт GitHub

```bash
docker version && k3d version && kubectl version --client && helm version
```

---

## 4. Запуск

### Шаг 1. Создать кластер k3d с метками узлов

1 server + 3 agents. Метки распределяют нагрузку и дают узел с «быстрым диском» для
демонстрации soft-affinity.

```bash
k3d cluster create lab \
  --servers 1 --agents 3 \
  --port "8081:80@loadbalancer" \
  --k3s-node-label "workload=system@agent:0" \
  --k3s-node-label "workload=app@agent:1" \
  --k3s-node-label "disk=fast@agent:1" \
  --k3s-node-label "workload=app@agent:2"

# проверка меток
kubectl get nodes --show-labels
```

> Узлы: `agent:0` = system (postgres, minio); `agent:1` = app + fast (предпочтительно для
> message-service); `agent:2` = app.

### Шаг 2. Установить S3-CSI драйвер (Helm)

```bash
helm repo add yandex-s3 https://yandex-cloud.github.io/k8s-csi-s3/charts
helm repo update
helm install csi-s3 yandex-s3/csi-s3 \
  --namespace kube-system \
  -f infra/csi-s3/values.yaml

kubectl -n kube-system get pods | grep -i s3   # должны быть Running
```
Подробности — [infra/csi-s3/README.md](infra/csi-s3/README.md).

### Шаг 3. Установить Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```
`--server-side` нужен, чтобы не упереться в лимит аннотации на большом CRD ApplicationSet.
Подробности — [infra/argocd/README.md](infra/argocd/README.md).

### Шаг 4. Запушить репозиторий в GitHub

```bash
git add .
git commit -m "k8s lab: kustomize + s3 csi + argocd"
git remote add origin https://github.com/<YOUR_USER>/<YOUR_REPO>.git
git push -u origin main
```

### Шаг 5. Подставить repoURL и применить Application

В `argocd/application-dev.yaml` замените `repoURL` на адрес вашего репозитория, затем:

```bash
kubectl apply -f argocd/application-dev.yaml
# одновременно держим только ОДИН overlay (см. docs/report.md → troubleshooting)
```

Дальше Argo CD сам склонирует репозиторий, прогонит kustomize и развернёт приложение.

