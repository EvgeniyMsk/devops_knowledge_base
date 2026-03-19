# 10. Helm в production

---
Уровень **Senior** — осознанно управлять **версиями chart’а**, не ломать потребителей при **обновлении values**, обеспечивать **обновление без простоя** на стороне Kubernetes и понимать, что **канареечный** / **blue-green** трафик делается **workload’ами и сетью**, а Helm лишь **доставляет** манифесты. Ниже — semver, обратная совместимость, rolling update, сценарий **rollback** и чеклист.

---

## Версионирование чарта

| Поле | Смысл |
|------|--------|
| **`version`** в `Chart.yaml` | Версия **упаковки** (chart): растёт при изменении шаблонов, values по умолчанию, зависимостей |
| **`appVersion`** | Ориентир на версию **приложения** / основного образа |

Ориентир **semver** для `version`:

- **MAJOR** — ломающие изменения в values или удаление ресурсов;
- **MINOR** — новые optional-параметры, новые optional ресурсы;
- **PATCH** — исправления шаблонов без изменения контракта.

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
version: 1.4.2
appVersion: "2.8.0"
```

Best practice: в **changelog** chart’а (или релиз-ноты) перечислять **breaking** изменения и миграции values.

---

## Обратная совместимость values

| Практика | Зачем |
|----------|--------|
| **Новые ключи** с **умолчаниями** в `values.yaml` | Старые `-f` не ломаются |
| **Не удалять** ключи в одном релизе без **deprecation** периода | Потребители umbrella-chart’ов |
| **`values.schema.json`** | JSON Schema валидация при `helm install` (Helm 3) |
| **`README`** | Явные «обязательные» и «удалено в vX» |

Пример мягкого переименования: поддерживать старый ключ через **`coalesce`**, новый — приоритетнее:

```yaml
# шаблон (идея)
{{- $replicas := .Values.replicaCount | default .Values.legacyReplicaCount | default 1 }}
```

---

## Обновление без простоя (zero-downtime)

Helm обновляет **Deployment** — за поведение отвечает **стратегия выката** и **пробы**.

### `RollingUpdate` с сохранением доступности

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      containers:
        - name: app
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

Смысл: **readiness** снимает Pod с **Service**, пока приложение не готово; **`maxUnavailable: 0`** не уменьшает число «готовых» ниже `replicas` при нормальном выкате.

Дополнительно в production:

- **`PodDisruptionBudget`** (`minAvailable` / `maxUnavailable`) для **voluntary** disruptions;
- достаточные **requests/limits** и **HPA**, если нагрузка меняется;
- **прогрев** кэшей в readiness или отдельный init.

---

## Canary и blue-green «через Kubernetes»

Helm **не** переключает процент трафика сам по себе. Типичные варианты:

| Подход | Идея |
|--------|------|
| **Два Deployment** + общий селектор или **Argo Rollouts** / **Flagger** | Постепенное увеличение веса «канарейки» |
| **Blue-green** | Два набора Pod’ов, переключение **Service** или **Ingress** на «зелёный» набор |
| **Service mesh** (Istio, Linkerd) | **VirtualService** / **TrafficSplit** с весами |

Helm поставляет CRD/манифесты; **политика трафика** живёт в конфигурации mesh или rollout-контроллера.

---

## Практика: обновить приложение и откатить при ошибке

### 1. Выкат с ожиданием готовности

```bash
helm upgrade my-app ./chart -n production \
  -f values-prod.yaml \
  --set image.tag=v2.9.1 \
  --wait \
  --timeout 15m
```

### 2. Если релиз неконсистентен — откат по истории

```bash
helm history my-app -n production
helm rollback my-app "$PREV_REVISION" -n production --wait --timeout 10m
```

### 3. Комбинация с **atomic** (автооткат при провале в окне timeout)

```bash
helm upgrade my-app ./chart -n production \
  -f values-prod.yaml \
  --set image.tag=v2.9.1 \
  --atomic --wait --timeout 15m
```

Комментарий: **atomic** не заменяет **наблюдаемость** (метрики ошибок после выката); для критичных сервисов добавьте **canary** или **ручной gate**.

---

## Production checklist (Senior)

| # | Вопрос |
|---|--------|
| 1 | Есть ли **changelog** chart’а и политика semver? |
| 2 | Новые values **optional** и задокументированы? |
| 3 | **Probes** + **RollingUpdate** проверены под нагрузочным тестом? |
| 4 | **PDB** и **HPA** согласованы с выкатом? |
| 5 | Runbook: **helm rollback** + критерий «откат не спасает» (миграции БД) |

---

## Дополнительные материалы

- [Charts — versions](https://helm.sh/docs/topics/charts/#the-chartyaml-file)
- [Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)

