# 3. Работа с values

---
**Values** — это входные параметры chart’а: базовые значения лежат в **`values.yaml`**, а для **dev / stage / prod** обычно подключают **отдельные файлы** и при необходимости **добивают** из CI через **`--set`**. Ниже — **порядок приоритета** (что перекрывает что), практики для production и готовые фрагменты **`values-dev.yaml`** / **`values-prod.yaml`**.

---

## Приоритет значений (от слабого к сильному)

| Источник | Комментарий |
|----------|-------------|
| **`values.yaml` в chart** | Значения по умолчанию в репозитории |
| **Values родительского chart’а** | Если chart подключён как **subchart** / dependency |
| **Файлы `-f` / `--values`** | Слева направо: **правый файл побеждает** при совпадающих ключах |
| **`--set` / `--set-string` / `--set-file`** | Самый высокий приоритет; удобно из CI для тега образа |

Официально порядок объединения описан в документации Helm: итог — **глубокий merge** словарей, на последних шагах перекрываются листовые поля.

```bash
helm upgrade --install my-app ./my-app \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag="sha-abc123"
```

Здесь **`values-prod.yaml`** перекроет совпадающие ключи из **`values.yaml`**, а **`--set image.tag`** перекроет и их.

---

## `--set` и `--set-string`

| Флаг | Когда |
|------|--------|
| **`--set key=value`** | Числа и булевы распознаются по типу; строка `"true"` может стать bool — см. документацию |
| **`--set-string key=value`** | Принудительно **строка** (часто для semver/`0644`) |
| **`--set-file key=path`** | Подставить содержимое файла (TLS, скрипты) |

```bash
helm template rel ./my-app -f values.yaml \
  --set-string image.tag="1.4.0-rc.1" \
  --set replicaCount=1
```

Production: сложные структуры из **десятков** `--set` неудобны — лучше **один** `-f env.yaml` + один-два `--set` из pipeline.

---

## Окружения: dev / stage / prod

| Подход | Плюсы | Внимание |
|--------|--------|----------|
| **`values-<env>.yaml` в Git** | Прозрачно, review в MR | **Секреты** не класть в открытый файл |
| **Общий `values-common.yaml`** + маленький `values-prod-delta.yaml` | Меньше дублирования | Явный порядок `-f` |
| **Внешние секреты** (SOPS, Vault Agent, External Secrets) | Безопасность | Сборка финального values в CI |

Рекомендация: в `values-prod.yaml` держать **жёсткие** `resources`, `replicaCount`, `PodDisruptionBudget`, а в **`values-dev.yaml`** — минимум реплик и ослабленные лимиты **осознанно**.

---

## Практика: `values-dev.yaml` и `values-prod.yaml`

Ниже — **фрагменты** одной и той же схемы ключей, что в разделе про шаблоны (можете адаптировать под свой chart).

### `values-dev.yaml`

```yaml
replicaCount: 1

image:
  repository: ghcr.io/myorg/my-app
  tag: dev
  pullPolicy: Always

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 250m
    memory: 256Mi

ingress:
  enabled: true
  className: nginx
  host: my-app.dev.example.com
```

### `values-prod.yaml`

```yaml
replicaCount: 3

image:
  repository: ghcr.io/myorg/my-app
  tag: ""  # задаётся CI: --set image.tag=$RELEASE_SHA
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 1Gi

ingress:
  enabled: true
  className: nginx
  host: my-app.example.com

podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

Комментарий: блок **`podDisruptionBudget`** должен быть **реализован в шаблонах** chart’а; если его нет — добавьте шаблон или используйте политики платформы.

### Рендер и установка

```bash
# Проверка до деплоя
helm template my-app ./my-app \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag="v1.4.2"

helm upgrade --install my-app ./my-app -n production \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag="v1.4.2" \
  --atomic --timeout 5m
```

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | Порядок `-f` задокументирован в README chart’а |
| 2 | В prod **не** хранить пароли в `values-prod.yaml` без шифрования |
| 3 | Один источник истины для **образа** (tag из CI) |
| 4 | `helm template` / dry-run с **теми же** `-f`, что и prod |
| 5 | Различия env **минимальны** и осознанны (не «copy-paste всего файла») |

---

## Дополнительные материалы

- [Values files](https://helm.sh/docs/chart_template_guide/values_files/)
- [Helm install — values](https://helm.sh/docs/intro/using_helm/#customizing-the-chart-before-installing)

