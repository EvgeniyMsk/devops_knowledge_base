# 10. Архитектура репозиториев GitOps

---
Очень важный, часто недооцененный блок: как организовать репозитории и вложить структуру так, чтобы управление несколькими приложениями оставалось управляемым, проверяемым и безопасным.
---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Monorepo** | Один репозиторий для множества приложений/компонентов. |
| **Multirepo** | Отдельные репозитории для разных приложений/команд. |
| **App-of-apps** | Паттерн, где существует “root” (корневое) приложение Argo CD, синхронизирующее манифесты “child”-приложений. |
| **Root application** | Приложение, которое применяет/обновляет набор child Applications (или манифесты, которые их создают). |
| **Child application** | Приложение Argo CD, которое управляет конкретным Kubernetes-набором ресурсов (Deployment/Service/CRDs и т.п.). |
| **Repository contract** | Согласованный “контракт” структуры репозитория: где лежат манифесты, как выбирается environment, какие переменные ожидаются. |

---

## Monorepo vs multirepo: как выбрать

### Monorepo — когда это выигрышно

Подходит, когда:

- команды готовы совместно поддерживать единый стандарт структуры (repository contract),
- важно простое атомарное изменение набора компонентов,
- релизы/зависимости между приложениями часто идут вместе,
- требуется единый pipeline контроля качества (lint/test/build) с понятной промо-логикой.

Риски:

- широкий blast radius (ошибка может затронуть больше приложений),
- сложнее разграничивать доступы по репозиториям (нужно делать это на уровне Argo/AppProject и RBAC).

### Multirepo — когда это выигрышно

Подходит, когда:

- у команд разные циклы поставки и ownership,
- нужна сильная изоляция (и по CI/CD, и по доступам),
- критична независимость версий/релизов.

Риски:

- больше “интеграционной” работы (сборка env-оверлеев, консистентность стандартов),
- сложнее управлять межкомпонентными релизами и зависимостями.

### Production best practice (независимо от формата)

- задайте и документируйте `repository contract`,
- используйте `AppProject` для per-app периметра (что разрешено откуда/куда),
- стремитесь к “маленьким” приложениями Argo CD (меньше ответственности на один child app).

---

## App-of-apps pattern: идея и назначение

Если у вас 10, 50 или 200 child applications, то руками управлять каждым становится трудно. App-of-apps решает это так:

- root application держит “каталог” child applications,
- child applications продолжают отвечать за конкретный workload,
- изменения набора приложений делаются через Git так же, как и изменения самих workload.

Типовая схема в репозитории:

- `apps/` — каталог child Applications (YAML),
- `apps/<env>/` — окружения (dev/stage/prod) или overlays,
- `clusters/` — опционально: конфигурации для разных кластеров.

---

## Пример: root application (управляет child Applications)

Идея: root application синхронизирует директорию с манифестами child `Application` CR.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-applications
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://git.example.com/platform/gitops.git
    targetRevision: main
    path: apps
    directory:
      recurse: true # позволяет root находить много child Application манифестов в поддиректориях
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd # где создаются child Applications (обычно argocd)
  syncPolicy:
    automated:
      prune: true      # важно только если набор child apps управляется исключительно из этого каталога
      selfHeal: true   # если манифесты child apps “сломались” — root вернет их в желаемое состояние
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true # снижает риск “невалидного порядка” при удалении ресурсов
```

Production нюанс:

- если вы включаете `prune: true` на root, убедитесь, что каталог `apps/` является полной картиной для всех child applications, иначе можно случайно удалить “лишнее”.

---

## Пример: child application (управляет конкретным workload)

Например, child application для `orders-api`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: orders-api
  namespace: argocd
spec:
  project: prod-orders # AppProject ограничивает периметр
  source:
    repoURL: https://git.example.com/services/orders-api.git
    targetRevision: v1.2.3
    path: deploy/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true    # уборка “мертвых” ресурсов при изменениях манифестов
      selfHeal: true
```

Комментарий по production:

- фиксируйте `targetRevision` на релизных тегах или git sha в production (не `main`), чтобы избежать неожиданных обновлений.

---

## Пример: AppProject как “периметр” (guardrails)

AppProject ограничивает, какие репозитории и destinations разрешены child applications.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: prod-orders
  namespace: argocd
spec:
  sourceRepos:
    - 'https://git.example.com/services/*' # разрешаем только нужные сервисы
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
  # Дополнительно можно ограничивать разрешенные cluster-scoped ресурсы через resourceWhitelist.
```

---

## Практика: реализовать root application и child apps

Шаги (минимально необходимое):

1) Создайте в `apps/` каталог манифестов child `Application` CR.
2) Разверните root application (он создаст/обновит child applications).
3) Проверьте, что child приложения получают желаемое состояние и у них health “green”.

Команды:

```bash
argocd app get root-applications

# убедиться, что child приложения создались/обновились
argocd app list | rg "orders-api|payments-api"
```

Production debug:

- если child application не появляется, сначала проверьте, что root application синхронизировал каталог `apps/` (sync status + events),
- затем проверьте RBAC: child Applications должны создаваться в namespace, куда у root есть доступ.

---

## Best practices: production checklist

| Практика | Зачем |
|----------|-------|
| Четкий `repository contract` | Меньше “магии” и ручных правок. |
| Root app управляет только набором child apps из одного каталога | Безопаснее `prune`. |
| `AppProject` как периметр | Снижение риска “не туда применили”. |
| Production фиксирует версии через tags/sha | Предсказуемые релизы. |
| Минимизируйте “ответственность” на один child app | Быстрее расследования и изменения. |

