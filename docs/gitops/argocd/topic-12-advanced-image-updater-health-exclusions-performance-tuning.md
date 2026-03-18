# 12. Advanced (для уверенного Middle+)

---

Если вы уже уверенно работаете с базовыми сущностями Argo CD, то следующий уровень обычно упирается в четыре практики production-эксплуатации:

- автoобновление образов (Image Updater),
- корректное определение health для сложных ресурсов (custom health checks),
- исключение шумных/внешних ресурсов из diff/health (resource exclusions),
- снижение нагрузки на контроллеры при росте количества приложений (performance tuning).

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Image Updater** | Компонент, который мониторит доступные образы (tags/digests) и обновляет значения в Git/Argo CD, чтобы заменить image в манифестах. |
| **CRD** | Custom Resource Definition: расширение Kubernetes схемы для ресурсов вроде `ImageUpdater`. |
| **Custom health check** | Lua-логика в `argocd-cm`, которая переопределяет health для конкретных Kubernetes-ресурсов/CRD. |
| **Resource exclusions** | Правила, которые исключают определенные ресурсы из diff/health/сверки, чтобы уменьшить шум и лишние пересчеты. |
| **Performance tuning** | Настройка параметров контроллера/репо-сервера для уменьшения очередей и ускорения reconciliation на больших инсталляциях. |

---

## Image Updater: автoобновление образов в production

### Зачем

Вместо ручного обновления `image: ...:tag` вы получаете управляемое обновление:

- по стратегии (semver/digest/newest),
- с ограничениями (allow/ignore tags),
- и с безопасной записью изменений обратно (git/Argo CD write-back).

### Production best practices

- для production используйте предсказуемые стратегии (например `semver` или `digest`),
- не обновляйте “вслепую” mutable tags (типа `latest`) без ограничений,
- включайте только необходимый scope (namePattern/label селекторы),
- фиксируйте релизную политику в Git (write-back в Git или в Argo CD в согласованном формате).

### Пример: ImageUpdater CRD (semver, scope на prod)

```yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: prod-image-updater
  namespace: argocd
spec:
  # Общие настройки по умолчанию для всех выбранных приложений
  commonUpdateSettings:
    updateStrategy: "semver"
    forceUpdate: false

  # Как компонент записывает обновления обратно
  writeBackConfig:
    method: "argocd" # альтернативно: method: "git" с gitConfig

  applicationRefs:
    - namePattern: "production-*"
      images:
        - alias: "api"
          imageName: "registry.example.com/orders-api:1.x" # semver range
```

Комментарий:

- `namePattern` должен соответствовать имени Argo CD Application (например `production-orders-api`).
- `alias` должен быть согласован с тем, как вы описываете соответствие image-значений в манифестах (в настройках маппинга Image Updater).

---

## Custom health checks: когда health “зелёный”, но фактически нет

### Зачем

Обычно Argo CD health неплохо работает для “типовых” ресурсов. Но для:

- сложных rollout’ов,
- CRD с нестандартными полями,
- случаев, когда default health слишком мягкий,

необходим custom health.

### Production best practices

- делайте health check deterministic: одинаковый status -> одинаковый health,
- проверяйте именно те поля, которые отражают бизнес-готовность (Ready/Available/conditions),
- возвращайте `Progressing`, а не `Healthy`, пока выполняется roll/update.

### Пример: health для Deployment по condition Ready

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # key формат: resource.customizations.health.<api-group>_<kind>
  # для Deployment core group: apps_Deployment
  resource.customizations.health.apps_Deployment: |
    hs = {}

    -- condition-based health: ищем Ready=True
    if obj.status ~= nil and obj.status.conditions ~= nil then
      for i, condition in ipairs(obj.status.conditions) do
        if condition.type == "Ready" and condition.status == "True" then
          hs.status = "Healthy"
          return hs
        end
      end
    end

    -- если Ready ещё не пришёл, но ресурс существует - считаем, что обновляется/создаётся
    if obj.metadata ~= nil then
      hs.status = "Progressing"
      hs.message = "Deployment is not Ready yet"
      return hs
    end

    hs.status = "Missing"
    return hs
```

Комментарий:

- Lua получает `obj` (Kubernetes resource) и возвращает таблицу с `hs.status` и опциональным `hs.message`.
- Для CRD/сложных ресурсов вы меняете критерии на то, что соответствует вашей готовности (например conditions по типам, phase, availableReplicas).

---

## Resource exclusions: меньше шума в diff и health

### Зачем

В реальных кластерах часть ресурсов:

- постоянно меняется внешними компонентами (events, endpoint slices),
- не относится к deployment-контракту сервиса,
- или не должна влиять на drift/diff.

Именно поэтому в production часто применяют exclusions.

### Глобальные exclusions (resource.exclusions в argocd-cm)

Пример: исключаем events и endpoint-related шум:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.exclusions: |
    - apiGroups:
        - "events.k8s.io"
      kinds:
        - Event
      clusters:
        - "*"

    - apiGroups:
        - "discovery.k8s.io"
      kinds:
        - EndpointSlice
      clusters:
        - "*"
```

### Альтернатива: ignoreDifferences на уровне Application

Если ресурс важен, но вы хотите игнорировать конкретное поле, то лучше использовать `ignoreDifferences`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: orders-api
  namespace: argocd
spec:
  # Игнорируем replicas, если они управляются HPA
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

Production best practice:

- исключайте/игнорируйте только то, что реально не должно считаться дрейфом вашего Git-контракта.

---

## Performance tuning: как не “задохнуться” при росте

### Зачем

Когда приложений становится много, возникают очереди:

- пересчёта health/sync,
- операций sync,
- генерации манифестов в repo-server.

В production tuning обычно начинают с масштабирования controller/repo-server и настройки concurrency/cache.

### Production best practices

- меняйте один параметр за раз и контролируйте эффект (очередь/latency),
- для “тяжёлых” Helm charts увеличивайте таймауты repo-server (если возникают timeouts),
- увеличивайте concurrency только при наличии CPU/memory и после измерений.

### Пример: правки в argocd-cmd-params-cm

Ниже только ключевые поля, которые чаще всего используются для оптимизации:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  # сколько одновременно пересчитывать status/health
  controller.status.processors: "50"

  # сколько одновременно выполнять sync operations
  controller.operation.processors: "25"

  # кэш состояния приложения (уменьшает лишние перегенерации)
  controller.app.state.cache.expiration: "2h0m0s"

  # лимит параллельных kubectl fork/exec (критично для кластера-агента)
  controller.kubectl.parallelism.limit: "50"

  # ограничение параллелизма генерации манифестов в repo-server
  reposerver.parallelism.limit: "5"
```

Комментарий по production:

- многие изменения в `argocd-cmd-params-cm` требуют перезапуска pod’ов (или хотя бы обновления конфигурации) — планируйте изменение как отдельный релиз.
- начните с умеренных значений и повышайте до “устойчивых очередей”, а не до абсолютного минимума latency (иначе вы просто загрузите кластер).

### Мини-практика проверки эффекта

```bash
# Проверка health/sync очереди (идея: смотреть метрики/очереди вашего мониторинга)
kubectl -n argocd get pods -o wide | rg "argocd-application-controller|argocd-repo-server"

# Посмотрите, не растет ли очередь операций (зависит от вашей системы мониторинга)
# Если есть Prometheus/Grafana - добавьте dashboard по очередям/латентности.
```

---

## Production checklist

| Что улучшить | Когда важно |
|--------------|--------------|
| Image Updater с semver/digest и ограниченным scope | Когда релизы частые и вы хотите уменьшить ручные изменения. |
| Custom health check для нестандартных CRD | Когда health “зелёный”, но SLO нарушаются/нет бизнес-готовности. |
| Resource exclusions/ignoreDifferences | Когда diff шумный или внешние компоненты постоянно меняют поля. |
| Performance tuning concurrency/cache | Когда число приложений/Helm генерации растет и появляются очереди. |

