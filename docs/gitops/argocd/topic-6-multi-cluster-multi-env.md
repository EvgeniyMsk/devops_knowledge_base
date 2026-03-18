# 6. Multi-cluster и multi-env

---
Эта тема — ключевой production‑навык: как управлять множеством Kubernetes‑кластеров и несколькими окружениями (dev/staging/prod) из Argo CD, при этом сохраняя дисциплину, воспроизводимость и безопасность.

Фокус: один repo → несколько environments, разные `values` для Helm, подключение внешних кластеров и базовая организация App’ов.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Cluster connection** | Подключение к внешнему кластеру из Argo CD (kubeconfig/credentials). |
| **Destination** | Где Argo CD применяет манифесты: `server` + `namespace`. |
| **Environment** | Логическое окружение (dev/staging/prod), которое обычно соответствует разным values, кластерам и namespace’ам. |
| **Single repo, multi env** | Подход, когда один Git‑репозиторий содержит манифесты/overlays и генерирует деплой для разных окружений. |
| **Helm values** | Параметры Helm‑чарта, которые меняются между окружениями (dev vs prod). |
| **AppSet (опционально)** | Автоматизация создания Application’ов для набора кластеров/окружений (через генераторы). |

---

## Подключение внешних кластеров (cluster registration)

Production best practice:

- подключайте кластеры строго через Argo CD credentials (не храните kubeconfig “вручную” в приложениях);
- минимизируйте права service account/пользователя, под которым Argo CD применяет манифесты.

### Концептуальная схема

1. В Argo CD добавляется cluster connection (secret/credentials).
2. В `Application.spec.destination.server` указывается нужный `server`.

Пример: указать destination на внешний cluster (логика зависит от того, как вы зарегистрировали cluster).

```yaml
spec:
  destination:
    server: https://cluster-1.example.com
    namespace: production
```

Комментарий:

- exact имя/URI `server` должен соответствовать записи cluster connection в Argo CD.
- лучше стандартизировать мэппинг “cluster-name → server url” в документации проекта.

---

## Разделение окружений: dev / staging / prod

Production best practice:

- разные окружения должны иметь явные границы: разные namespace’ы и/или разные кластеры;
- разные окружения должны иметь разные значения конфигурации (Helm values/overlays).

Типичный подход:

- dev: небольшие ресурсы, чаще обновления, чаще auto-sync
- staging: production-like, больше тестов, auto-sync по решению команды
- prod: минимальные “права на изменение”, ручной approve/контроль деплоя

---

## Один repo → несколько environments

### Вариант A: отдельные values-файлы для Helm

Обычно в репозитории хранят:

- `charts/myapp`
- `environments/dev/values.yaml`
- `environments/staging/values.yaml`
- `environments/prod/values.yaml`

Пример Application для dev с отдельными values:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://git.example.com/platform/apps.git
    targetRevision: main
    chart: myapp
    helm:
      valueFiles:
        - environments/dev/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

Production best practice:

- для prod часто отключают automated, чтобы деплой был управляемым;
- держите `targetRevision` фиксированным тегом/sha для “воспроизводимого релиза”.

---

### Вариант B: Kustomize overlays (dev/stage/prod)

Если вы используете Kustomize:

- `overlays/dev`, `overlays/staging`, `overlays/prod`
- в Application указываете `source.path`.

Пример:

```yaml
source:
  repoURL: https://git.example.com/platform/k8s-manifests.git
  targetRevision: main
  path: apps/myapp/overlays/prod
```

Best practice:

- overlay должен быть “полным” для окружения: не храните нужные параметры “где-то по пути”, иначе трудно восстановить желаемое состояние.

---

## Управление созданием Application’ов (масштабирование)

Когда environment’ов и кластеров много, вручную писать Application’ы становится дорого.

Production best practice:

- используйте AppSet (если хотите автоматизировать),
- либо генерируйте Application’ы через templates/include в Git.

Короткая концепция AppSet:

- генератор списка окружений → для каждого создаётся Application

В этом репозитории конкретная реализация AppSet зависит от выбранного подхода, поэтому здесь оставляем “архитектурный” шаблон.

---

## Best practices: production чеклист

- **Границы окружений**: разные namespace’ы и/или разные кластеры; prod не смешивать с dev.
- **Разные values**: dev/staging/prod имеют разные параметры (реплики, ресурсы, ingress, секреты).
- **Подключение кластеров**: cluster connection зарегистрирован безопасно, с минимальными правами.
- **Релизоподобность**: фиксируйте `targetRevision` и/или используйте structured tags.
- **Контроль деплоя в prod**: manual sync/approve, осторожная selfHeal/prune стратегия.
- **Документируйте мэппинг**: cluster-name → destination.server; environment → path/values.

---

## Дополнительные материалы

- [Argo CD — Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [Argo CD — Applications](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [Argo CD — ApplicationSet](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
