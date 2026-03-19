# 7. Лучшие практики чартов

---
Устойчивые **чарты** в **production** опираются на **соглашения об именах**, предсказуемые **лейблы и аннотации**, вынос общей логики в **`templates/_helpers.tpl`** и принцип **DRY** (не копировать одно и то же в десять файлов). Ниже — ориентиры Helm/Kubernetes, примеры **`define` / `include`** и чеклист.

---

## Именование

| Объект | Практика |
|--------|----------|
| **Chart name** | Короткое имя в **нижнем регистре**, без пробелов (как в `Chart.yaml` → `name`) |
| **Ресурсы в кластере** | Стабильный префикс **`Release.Name`** или **`fullname`** (см. helper ниже), обрезка до **63** символов для имён DNS |
| **Шаблоны файлов** | `deployment.yaml`, `service.yaml` — одна сущность Kubernetes на файл, когда возможно |

Рекомендации Helm: [conventions](https://helm.sh/docs/chart_best_practices/conventions/).

---

## Лейблы и аннотации

### Рекомендуемые лейблы (согласованность с экосистемой)

Используйте метки из **Kubernetes** / **Helm** [recommended labels](https://helm.sh/docs/chart_best_practices/labels/):

| Лейбл | Назначение |
|-------|------------|
| `app.kubernetes.io/name` | Логическое имя приложения |
| `app.kubernetes.io/instance` | Уникальный идентификатор релиза (`Release.Name`) |
| `app.kubernetes.io/version` | Версия приложения **или** образа (из `appVersion` / values) |
| `app.kubernetes.io/managed-by` | Обычно **`Helm`** (`{{ .Release.Service }}`) |
| `helm.sh/chart` | `chartname-version` для связи с chart’ом |

### Аннотации для отслеживания Helm

Полезно на **Deployments/StatefulSets** (если не генерирует ваш инструмент автоматически):

```yaml
meta.helm.sh/release-name: {{ .Release.Name }}
meta.helm.sh/release-namespace: {{ .Release.Namespace }}
```

---

## `_helpers.tpl`: переиспользуемые фрагменты

Файлы **`templates/*.tpl`** не рендерятся как отдельные манифесты — в них объявляют **именованные шаблоны** через **`define`** и подключают через **`include`**.

### Пример `templates/_helpers.tpl`

```yaml
{{/*
Полное имя релиза (короткий учебный вариант; в `helm create` — более полная логика).
*/}}
{{- define "my-app.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{/*
Имя chart для лейбла helm.sh/chart
*/}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" -}}
{{- end }}

{{/*
Общий набор лейблов
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Использование в `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

**`include`** возвращает строку; **`nindent`** добавляет отступ — типичный **DRY**-паттерн.

---

## DRY: что выносить в helpers

| Повтор | Куда |
|--------|------|
| Одинаковые **лейблы** на Deployment/Service/Ingress | `define "chart.labels"` |
| **fullname** / **service account name** | отдельный `define` |
| **Образ** `repository:tag` | опционально `define "chart.image"` |
| Сложный **probe** или **securityContext** | `define "chart.defaultSecurityContext"` + `nindent` |

Не превращайте helpers в «всё в одном файле на 500 строк» — группируйте по смыслу.

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | Лейблы **согласованы** с дашбордами и селекторами **NetworkPolicy** |
| 2 | **`app.kubernetes.io/version`** не содержит секретов и не ломает кардинальность |
| 3 | Имена ресурсов **уникальны** между релизами в одном namespace |
| 4 | `_helpers.tpl` покрыт **ревью** при изменении `Chart.Name` / переезде chart’а |
| 5 | Документация в **`README.md`**: обязательные values и пример `helm install` |

---

## Дополнительные материалы

- [Chart best practices](https://helm.sh/docs/chart_best_practices/)
- [Labels and annotations](https://helm.sh/docs/chart_best_practices/labels/)
- [Named templates](https://helm.sh/docs/chart_template_guide/named_templates/)

