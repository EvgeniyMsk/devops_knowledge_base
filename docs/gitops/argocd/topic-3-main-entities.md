# 3. Основные сущности

---
Эта тема помогает уверенно работать с ключевыми ресурсами Argo CD: **Application** и **AppProject**. Разберём, как описывать приложение (Helm chart и Kustomize), как настроить автоматическую синхронизацию, self-heal и prune, и какие production-best-practices помогут избежать “неожиданных” деплоев.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Argo CD** | GitOps-контроллер для Kubernetes, который поддерживает desired-state из Git. |
| **Application** | CRD Argo CD: описывает, какой репозиторий/путь/чарт нужно деплоить в какой cluster/namespace. |
| **AppProject** | CRD Argo CD, который ограничивает: какие репозитории разрешены, куда можно деплоить, какие ресурсы и роли допустимы. |
| **auto-sync** | Автоматическая синхронизация Application при изменениях в Git (если включена syncPolicy automated). |
| **self-heal** | Автоматическое исправление drift: если ресурс в кластере изменили вручную, Argo CD вернёт его к состоянию из Git. |
| **prune** | Удаление ресурсов, которые исчезли из desired-state (важно для чистоты и предсказуемости). |
| **Sync wave** | Порядок применения ресурсов (через Argo CD waves/annotations, зависит от настроек). |

---

## Application: Helm chart (пример)

Production best practice: ограничивайте namespace назначения через AppProject.

```yaml
# Application для Helm chart
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: orders-api
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://git.example.com/platform/helm-charts.git
    targetRevision: v1.8.3
    chart: orders-api
    helm:
      # values можно передавать явно (для production это удобно фиксировать версией)
      values: |
        image:
          tag: "1.2.3"
        replicaCount: 3
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      # Включайте carefully:
      # - selfHeal вернёт drift обратно к Git (полезно, но может “сломать” ручные hotfix).
      # - prune удалит ресурсы, которых больше нет в Git (обязательно для чистоты).
      selfHeal: true
      prune: true
    # CreateNamespace помогает не падать, когда namespace ещё не создан
    syncOptions:
      - CreateNamespace=true
```

Комментарий:

- `project: platform` — ключевой контроль доступа (см. AppProject ниже).
- `targetRevision` фиксируйте тегом или SHA, чтобы деплой был воспроизводимым.

---

## Application: Kustomize (пример)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://git.example.com/platform/k8s-manifests.git
    targetRevision: main
    path: apps/payments/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - Validate=true
```

Best practice:

- `path` должен быть “эквивалентом окружения” (overlay зафиксирован, а не произвольный набор файлов).
- `Validate=true` повышает безопасность за счёт проверки схем/форматов на стороне Argo CD.

---

## AppProject: ограничения для production

AppProject — это “контроль периметра”: отрежет случайные попытки деплоя не туда и не из тех репозиториев.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: "Platform project limits"
  sourceRepos:
    - https://git.example.com/platform/*
  destinations:
    - server: https://kubernetes.default.svc
      namespace: production
    - server: https://kubernetes.default.svc
      namespace: staging
  # В production часто дополнительно ограничивают разрешённые resource kinds
  # и делают более строгие политики.
  #
  # В зависимости от нужд команды можно включить cluster resource restrictions.
```

Комментарий:

- держите sourceRepos максимально узкими (regex/маски зависит от возможностей настройки).
- destinos ограничивайте конкретными namespace или кластерами.

---

## Настройка automated sync: когда включать и когда нет

Production подход:

- на staging — можно чаще включать auto-sync, чтобы быстрее видеть эффект изменений;
- на production — часто оставляют auto-sync выключенным и используют manual/approve.

Мини-пример: auto-sync только на staging.

```yaml
syncPolicy:
  automated:
    selfHeal: true
    prune: true
```

Если auto-sync выключен:

- Argo CD покажет drift, но не применит изменения автоматически.

---

## self-heal и prune: аккуратное включение

- **self-heal = true** полезен, если вы принципиально запрещаете ручные изменения в production.
- **prune = true** полезен, если вы считаете Git единственным источником правды и хотите чистить “хвосты”.

Best practice “первого включения”:

1. Включайте `prune` сначала в non-prod.
2. Убедитесь, что репозиторий содержит полный desired-state (иначе prune может удалить “лишнее” из кластера).
3. Потом переносите подход в production.

---

## Практика: проверить всё через UI и CLI

Команды:

```bash
argocd app list
argocd app get orders-api
argocd app sync orders-api
argocd app wait orders-api --health
```

Проверка production-best practice:

- перед синхронизацией убедитесь, что Argo CD показывает корректный diff (изменения только ожидаемые).
- используйте `app wait` чтобы pipeline/скрипт ожидал health, а не “верил в успех”.

---

## Дополнительные материалы

- [Argo CD — Application](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [Argo CD — AppProject](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)

