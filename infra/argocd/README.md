# Установка Argo CD (bootstrap)

Argo CD — GitOps-оператор: следит за вашим GitHub-репозиторием и приводит кластер к тому,
что описано в Git. Ставится один раз вручную.

## Установка

```bash
# 1) namespace для Argo CD
kubectl create namespace argocd

# 2) официальные манифесты установки
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3) дождаться готовности
kubectl -n argocd rollout status deploy/argocd-server
```

## Доступ к веб-интерфейсу

```bash
# пробрасываем порт на локальную машину
kubectl -n argocd port-forward svc/argocd-server 8080:443

# логин admin; стартовый пароль:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Откройте https://localhost:8080 (логин `admin`).

## Подключение приложения

После того как репозиторий с манифестами запушен в GitHub и в
`argocd/application-*.yaml` подставлен ваш `repoURL`:

```bash
kubectl apply -f argocd/application-dev.yaml
# при необходимости и prod:
# kubectl apply -f argocd/application-prod.yaml
```

Дальше Argo CD сам склонирует репозиторий, прогонит kustomize для overlay'я и развернёт
приложение. Цель — статус **Synced / Healthy**.

> Приватный репозиторий? Сначала добавьте доступ:
> `argocd repo add https://github.com/<user>/<repo>.git --username <user> --password <token>`
> (или через Settings → Repositories в UI).
