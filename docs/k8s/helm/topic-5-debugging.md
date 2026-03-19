# 5. Отладка Helm

---
Чарт «не ставится» или ставится **частично** — типичный рабочий сценарий. Умение по шагам отделить ошибку **шаблона**, **values**, **валидации chart’а** и **отказа API Kubernetes** сильно экономит время. Ниже — **`helm template`**, **`helm lint`**, **`--dry-run --debug`** и учебные поломки для тренировки.

---

## `helm template` — рендер без кластера

Показывает итоговые манифесты и ловит ошибки **Go template** / **`required`**.

```bash
helm template rel ./my-chart -f values.yaml --debug
```

- **`--debug`** — подробный вывод: какие файлы отрендерены, иногда цепочка ошибок (`template: ...`).
- Для точного воспроизведения окружения используйте **те же** `-f` и `--set`, что в CI/prod.

Best practice: **`helm template` в CI** на каждый MR с чартом.

---

## `helm lint` — проверка структуры chart’а

```bash
helm lint ./my-chart
helm lint ./my-chart -f values-prod.yaml
```

Проверяет **`Chart.yaml`**, наличие **`values.yaml`**, частые антипаттерны и синтаксис шаблонов (на уровне линтера).

Комментарий: **`lint`** не заменяет **кластерную** проверку (CRD, admission webhook, квоты).

---

## `helm install` / `upgrade` с `--dry-run` и `--debug`

Запрос к **Kubernetes API** (валидация схемы, иногда **admission**), **без** применения ресурсов:

```bash
helm upgrade --install my-app ./my-chart -n apps \
  -f values.yaml \
  --dry-run \
  --debug
```

Что даёт:

- проверка, что **сервер** принимает объекты (в отличие от «только шаблон»);
- в выводе видны **хуки** и предупреждения API.

Для **server-side** проверки без изменений часто дополнительно прогоняют сгенерированный YAML через:

```bash
helm template rel ./my-chart -f values-prod.yaml | kubectl apply --dry-run=server -f -
```

(актуально, если в pipeline есть живой кластер и контекст).

---

## Куда смотреть при ошибке

| Слой | Типичный симптом | Действие |
|------|------------------|----------|
| **Шаблон** | `error calling template`, `nil pointer` | Проверить `.Values.*`, `if` вокруг опциональных блоков, отступы **`nindent`** |
| **values** | Пустой образ, неверный тип (`replicaCount` строкой) | `helm get values`, сравнить с `values.schema.json` если есть |
| **YAML** | `yaml: line X` после рендера | Лишние пробелы в `range`, забытый `-` у `{{-` |
| **API Kubernetes** | `Forbidden`, `Invalid`, webhook denied | `kubectl describe`, логи контроллера, политики **OPA/Gatekeeper** |

---

## Практика: намеренно ломать чарт (лаборатория)

### 1. Сломать шаблон

- Удалить закрывающий `{{- end }}` у `range`.
- Обратиться к несуществующему ключу без `default`: `{{ .Values.oops.nested }}`.

Ожидание: **`helm template`** падает с указанием файла шаблона.

### 2. Сломать values

- Убрать обязательное поле, защищённое **`required`**, или передать **неверный тип**.

Ожидание: понятное сообщение `required "..." .Values...`.

### 3. Сломать манифест с точки зрения API

- Задать **`replicas: -1`** или **неверный** `apiVersion` под ваш кластер.

Ожидание: **`--dry-run=server`** или реальный `install` покажут ошибку валидации.

Fix: вернуть валидные поля, снова `helm template` → `kubectl apply --dry-run=server`.

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | В MR: **`helm lint` + `helm template`** с prod-like values |
| 2 | Перед релизом: **`--dry-run`** на stage с тем же **Kubernetes minor**, что prod |
| 3 | Хранить **минимальный** набор values для «smoke render» в CI |
| 4 | Ошибки webhook фиксировать **текстом** в runbook (какой политикой режется) |
| 5 | Не дебажить только `kubectl`; сравнивать с **`helm get manifest`** |

---

## Дополнительные материалы

- [helm template](https://helm.sh/docs/helm/helm_template/)
- [helm lint](https://helm.sh/docs/helm/helm_lint/)
- [helm install — dry run](https://helm.sh/docs/intro/using_helm/#dry-run)

