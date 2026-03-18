# 8. Production use cases

---
Эта тема — про “engineering thinking”: как применять Argo CD в реальных сценариях production, где важны безопасность релиза, наблюдаемость и контроль риска. Рассматриваем:

- canary deployments (часто в связке с Argo Rollouts),
- blue/green (похожая идея, но с переключением окружений/версий),
- feature environments / preview apps (для быстрого обзора изменений).

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Canary deployment** | Новая версия получает небольшой процент трафика/реплик, после чего долю увеличивают при успешных метриках. |
| **Blue/Green** | Два окружения: blue (стабильная) и green (новая). Переключение трафика происходит после проверки. |
| **Argo Rollouts** | Kubernetes-расширение для progressive delivery (canary/blue-green) с контролем шагов и анализов. |
| **Rollout resource** | Аналог Deployment в Argo Rollouts; управляет стратегией и версиями. |
| **AnalysisTemplate** | Описание того, какие метрики/проверки нужно выполнить перед продвижением canary. |
| **Preview app / feature environment** | Динамическое окружение для конкретной feature/MR: отдельные namespace/URL. |

---

## Canary deployments (production mindset)

### Почему canary лучше “всем сразу”

- меньше blast radius,
- быстрее получаете обратную связь от метрик,
- проще откатиться.

### Production best practice: canary + анализ метрик

Идея:

1) Argo CD деплоит Rollout (desired state из Git).
2) Argo Rollouts постепенно продвигает canary.
3) Перед увеличением доли делаются проверки (health + метрики).

Мини-пример Rollout (концептуально, под canary):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: orders-api
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      containers:
        - name: api
          image: registry.example.com/orders-api:1.2.3
  strategy:
    canary:
      steps:
        - setWeight: 10  # 10% трафика на canary
        - pause: { duration: 2m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

Комментарий:

- паузы и проценты должны быть “под ваши метрики”: сколько нужно, чтобы увидеть деградацию.
- откат Rollout обычно делается возвращением на предыдущую revision (через Argo CD sync на предыдущий tag или manual rollback).

---

## Blue/Green (в связке с progressive delivery)

Практический подход:

- blue/green проще “сделать вручную” через Service selector или Argo CD переключение,
- но лучше управлять progressive delivery через Rollouts — меньше ручных действий и больше предсказуемости.

Концептуально:

- green поднимается как отдельная версия,
- переключение трафика — после проверки.

В production best practice:

- заранее определить критерии “зелёного” релиза (metrics, error rate, latency, SLO),
- иметь rollback план.

---

## Feature environments / preview apps

### Production best practice: отдельные “миры” для MR

Вместо того чтобы релизить “вместе”, создайте preview environment, где:

- есть отдельный namespace,
- есть уникальный URL/hostname,
- ресурсы ограничены requests/limits,
- окружение автоматически чистится.

Пример naming strategy:

```text
namespace: review-$CI_MERGE_REQUEST_IID
hostname: $CI_MERGE_REQUEST_IID.review.example.com
```

Argo CD обычно поддерживает preview через отдельные Applications (Application per MR) либо через генерацию AppSet.

Мини-пример Application (conceptual):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: orders-api-review-123
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://git.example.com/platform/apps.git
    targetRevision: main
    path: apps/orders-api/overlays/review
  destination:
    server: https://kubernetes.default.svc
    namespace: review-123
```

Комментарий:

- в overlay `review` вы включаете небольшие лимиты ресурсов,
- интегрируете ingress/hostname.

---

## Best practices итог (что делать в production)

1. **Canary/blue-green**: используйте step-based rollout и pause по duration.
2. **Метрики**: обязательно связывайте promotion с health/метриками (через AnalysisTemplate или внешние проверки).
3. **Preview apps**: изолируйте окружения по MR и обязательно чистите их (иначе “кладбище” ресурсов).
4. **Откат**: откат должен быть воспроизводимым через Git (sync на предыдущий tag) или через rollback tool/команду.
5. **Ограничение ресурсов**: preview и canary не должны “съедать” production-ресурсы.

---

## Дополнительные материалы

- [Argo Rollouts — documentation](https://argoproj.github.io/argo-rollouts/)
- [Argo CD — Application](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
