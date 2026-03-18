# 4. Sync стратегии и управление состоянием

---
Уровень middle+: как в Argo CD управлять синхронизацией: manual vs auto, sync hooks (PreSync/PostSync), sync waves (порядок применения), health checks и стратегии rollback. Это ключ к безопасным деплоям без “случайных” изменений.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Manual sync** | Синхронизация по ручному запросу пользователя в UI/CLI. |
| **Auto sync** | Автоматическая синхронизация при изменениях в Git (через `syncPolicy.automated`). |
| **Sync policy** | Часть `Application.spec.syncPolicy`: управляет auto-sync, self-heal, prune и syncOptions. |
| **Sync hook** | Специальный ресурс, который выполняется до/после sync (например PreSync/PostSync), в зависимости от аннотаций. |
| **Sync wave** | Порядок применения ресурсов: ресурсы с разными `argocd.argoproj.io/sync-wave` применяются в заданной последовательности. |
| **Health** | Статус “здоровья” приложения/ресурсов, который Argo CD использует при ожидании успешного sync. |
| **Rollback** | Возврат к предыдущей ревизии Git и повторная sync вместо “ручных правок” в кластере. |

---

## Manual vs Auto sync: что выбирать в production

### Manual sync (часто для production)

Production best practice:

- production деплоить вручную (manual approve) либо через отдельный процесс;
- auto-sync включать сначала на staging/non-prod, где меньше риска.

### Auto sync (часто для staging/review)

Когда включаете auto-sync:

- `prune` и `selfHeal` добавляйте только после уверенности, что ваш Git действительно содержит full desired-state;
- убедитесь, что hooks идемпотентны (можно безопасно повторить).

Мини-пример Application (auto-sync):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: orders-api
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://git.example.com/platform/k8s-manifests.git
    targetRevision: main
    path: apps/orders-api
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

Комментарий:

- `Validate=true` помогает поймать ошибки манифестов до apply.

---

## Sync hooks: PreSync и PostSync

### Зачем hooks

Hooks нужны, когда нужно выполнить “операции вокруг” синхронизации:

- подготовка CRDs/каталогов/секретов;
- миграции или прогрев;
- post-deploy проверка или “переключение” (например, обновить ConfigMap, который используется приложением).

Best practice:

- hooks должны быть **идемпотентными**;
- не делайте hooks слишком “тяжёлыми” — они увеличивают длительность sync.

### Пример: PreSync hook (Job)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-sync-migration
  namespace: production
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    # Можно задавать “вес” выполнения, если используется больше одного hook.
    argocd.argoproj.io/hook-weight: "0"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myrepo/migrations:1.2.3
          command: ["./migrate.sh"]
```

### Пример: PostSync hook (smoke test)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: post-sync-smoke-test
  namespace: production
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: smoke
          image: curlimages/curl:8.6.0
          command: ["sh", "-c", "curl -fsS https://myapp.example.com/health"]
```

---

## Sync waves: порядок деплоя (CRD → приложение → тесты)

Когда Argo CD применяет ресурсы, важно задать последовательность:

1. CRDs (например, `apiextensions.k8s.io/v1`)
2. CR-ы (custom resources)
3. Деплой приложения
4. Тесты/пост-процессы

Пример:

```yaml
# CRD (применять первым)
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
  annotations:
    argocd.argoproj.io/sync-wave: "-10"
spec:
  # ...
```

```yaml
# Application CR (после CRD)
apiVersion: example.com/v1
kind: Widget
metadata:
  name: widget-1
  annotations:
    argocd.argoproj.io/sync-wave: "-5"
spec:
  # ...
```

```yaml
# Deployment (основной слой)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  # ...
```

Комментарий:

- значения `sync-wave` — строки, но это “числовая шкала”: меньше = раньше.
- если “волн” много — документируйте в репозитории, иначе через месяц никто не поймёт порядок.

---

## Health checks: как Argo CD считает success

Практика:

- всегда ждите health, а не просто “sync выполнен”.

CLI:

```bash
argocd app sync orders-api
argocd app wait orders-api --health --timeout=180
```

Production best practice:

- если нужно строгое определение “здоровья”, подключайте custom health checks (по необходимости) или следите за типовыми health-сигналами ресурсов (Deployment/Job/Service).

---

## Rollback сценарий

Production стратегия rollback обычно такая:

1) определить, к какой revision (commit) нужно откатиться
2) выполнить sync на предыдущую версию (желательно через CLI/UI)
3) не делать ручных правок “поверх” — иначе вернём drift

Примеры CLI:

```bash
# Посмотреть историю ревизий
argocd app history orders-api

# Откатиться к конкретной ревизии (идея)
argocd app rollback orders-api <REVISION>
```

Если rollback не настроен как команда (зависит от окружения/версии CLI), можно сделать:

```bash
argocd app sync orders-api --revision <REVISION>
```

---

## Production best practices (итог)

- production: чаще manual sync; staging/review: auto-sync по необходимости.
- hooks: идемпотентные, быстрые, с понятным delete-policy.
- sync waves: документируйте порядок (особенно CRD → CR → app).
- всегда используйте `argocd app wait --health` и таймауты.
- rollback делайте через Git ревизии (re-sync), а не через ручные правки в кластере.

---

## Дополнительные материалы

- [Argo CD — Sync hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/)
- [Argo CD — Sync waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [Argo CD — Health checks](https://argo-cd.readthedocs.io/en/stable/user-guide/health/)
- [Argo CD — Rollbacks](https://argo-cd.readthedocs.io/en/stable/user-guide/commands/#rollback)

