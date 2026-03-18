# 1. База: Концепции GitOps

---
Цель темы — понимать, **почему Argo CD вообще существует** и как GitOps работает на уровне принципов. Разберём GitOps как подход: pull vs push, desired state vs actual state и дрейф (drift detection). В конце — практика руками “идеальный деплой через Git” и моделирование drifа.

---

## Определения

| Термин | Определение |
|--------|-------------|
| **GitOps** | Подход к доставке: desired-state описывается в Git, а контроллеры (Argo CD) постоянно приводят кластер к состоянию из репозитория. |
| **Desired state** | Желаемое состояние системы (какие манифесты должны быть применены). |
| **Actual state** | Реальное состояние в кластере (что сейчас поднято). |
| **Drift (дрейф)** | Расхождение между desired state в Git и actual state в кластере (например, кто-то руками изменил ресурс). |
| **Drift detection** | Механизм выявления расхождений: контроллер сравнивает Git и фактическое состояние. |
| **Pull vs Push** | Pull: контроллер сам “тянет” desired-state из Git. Push: система “толкает” изменения наружу/в конфиг (в GitOps обычно принят pull). |
| **Sync (синхронизация)** | Приведение actual state к desired state (применение изменений). |
| **Reconcile (reconciliation loop)** | Цикл проверки/коррекции состояния: если есть drift — инициируется sync. |

---

## Pull vs Push в GitOps

В GitOps **pull** обычно означает:

- есть репозиторий с манифестами,
- контроллер (Argo CD) периодически или по событиям забирает изменения,
- далее сравнивает и применяет в кластер.

Production best practice:

- контроллер должен быть единственным источником “выполнения” деплоя (не делайте параллельно manual `kubectl apply` в production, иначе постоянно ловите drift).

---

## Desired state vs actual state: как это работает на практике

Упрощённо схема такая:

```text
Git (desired state) --(pull)--> Argo CD --(compare)--> drift?
                           |
                           +-- if drift: sync -> Kubernetes (actual state)
```

Для пользователя это выражается в UI/статусе приложения:

- `OutOfSync` — есть drift,
- `Synced` — cluster соответствует Git.

---

## Как моделировать drift руками (практика)

### Шаг 1: “идеальный деплой” через Git

1. Описываете ресурс в Git (например, Deployment/ConfigMap).
2. Делаете sync (через Argo CD — это и есть “деплой через Git”).

Мини‑пример манифеста ConfigMap:

```yaml
# config/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_MODE: "production"
```

### Шаг 2: сломать actual state (создать drift)

После того как Argo CD синхронизировал ресурс:

1. Вручную меняете ресурс в кластере (например, через `kubectl edit` или `kubectl patch`).
2. Argo CD увидит расхождение и переключит приложение в `OutOfSync`.

Пример “сломать” поля:

```bash
kubectl -n production patch configmap app-config \
  --type merge \
  -p '{"data":{"APP_MODE":"drifted-by-hand"}}'
```

---

## Production best practices для дрейфа

- **Не допускайте ручных изменений в production**: изменения должны идти через PR → merge в Git.
- Если ручные изменения нужны для отладки — делайте их в отдельном namespace/среде и документируйте, затем возвращайте состояние в Git.
- Следите за политиками sync:
  - либо manual sync (чёткий контроль),
  - либо автоматический sync с дополнительными защитами (reviews/approval в pipeline, ветки protected).

---

## Практика для уверенности (чеклист)

- Понимаете, что такое desired/actual и drift.
- Можете объяснить, почему pull удобнее для воспроизводимости.
- Можете смоделировать drift и объяснить, что вы ожидали увидеть в Argo CD.

---

## Дополнительные материалы

- [Argo CD — Concepts](https://argo-cd.readthedocs.io/en/stable/user-guide/intro/)
- [GitOps — Principles](https://opengitops.dev/)

