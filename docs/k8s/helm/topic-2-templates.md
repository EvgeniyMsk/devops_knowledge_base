# 2. Шаблоны Helm (Templates)

---
**Шаблоны** — ядро Helm: это **Go templates** (плюс библиотека **Sprig**), которые превращают `values.yaml` в готовые манифесты Kubernetes. Без уверенного владения **`{{ .Values }}`**, **`if` / `range`**, **`default` / `required`**, **`toYaml` + `nindent`** сложно дойти до уровня **Middle+**. Ниже — синтаксис, функции, пайплайны и минимальный **кастомный** chart (Deployment, Service, опционально Ingress).

---

## Контекст и `.Values`

В шаблоне доступны встроенные объекты:

| Объект | Содержимое |
|--------|------------|
| **`.Values`** | Содержимое `values.yaml` + переопределения `-f` / `--set` |
| **`.Release`** | `Name`, `Namespace`, `Service` и др. |
| **`.Chart`** | Поля из `Chart.yaml` (`Name`, `Version`, `AppVersion`, …) |
| **`.Capabilities`** | Версия API Kubernetes и т.д. |

```yaml
replicas: {{ .Values.replicaCount }}
```

---

## Условия и циклы

```yaml
{{- if .Values.ingress.enabled }}
# манифест Ingress
{{- end }}
```

```yaml
{{- range .Values.extraEnv }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}
```

Комментарий: **`{{-`** и **`-}}`** убирают лишние переносы строк в выводе — иначе YAML часто «разъезжается» или появляются пустые строки.

---

## Функции: `default`, `required`

**`default`** — запасное значение, если вход пустой или «zero value»:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

**`required`** — если значение не задано, **helm template / install** упадёт с понятной ошибкой (лучше, чем «пустой образ» в prod):

```yaml
repository: {{ required "Задайте .Values.image.repository" .Values.image.repository | quote }}
```

---

## `toYaml`, `indent`, `nindent`

Встраивание фрагментов YAML из values (ресурсы, nodeSelector, tolerations):

```yaml
resources:
  {{- toYaml .Values.resources | nindent 12 }}
```

- **`nindent N`** — вставить перенос + **N** пробелов у каждой строки (типично для `containers:` под **12** spaces).
- **`indent`** — без начального переноса; чаще удобнее `nindent`.

---

## Пайплайны (`|`)

Значение передаётся в функцию справа налево по цепочке:

```yaml
{{ .Values.name | upper | quote }}
```

```yaml
{{ .Values.podAnnotations | toYaml | nindent 8 }}
```

---

## Практика: фрагменты чарта

Ниже — **учебный** минимум. Имена могут совпадать с тем, что даёт `helm create`; при желании замените `my-release` на `{{ .Release.Name }}` и вынесите общие лейблы в `_helpers.tpl`.

### `values.yaml`

```yaml
replicaCount: 2

image:
  repository: ghcr.io/myorg/my-app
  tag: ""

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false
  className: nginx
  host: my-app.example.com
```

В корне чарта нужен **`Chart.yaml`** с полем **`appVersion`**, иначе подстановка `default .Chart.AppVersion` для тега образа не сработает осмысленно.

### `templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name }}
    spec:
      containers:
        - name: app
          image: "{{ required "image.repository обязателен" .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### `templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
      protocol: TCP
      name: http
```

### `templates/ingress.yaml` (опционально)

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

Проверка рендера:

```bash
helm template test-release . -f values.yaml --debug
```

---

## Production best practices

| Практика | Зачем |
|----------|--------|
| **`required`** для обязательных полей | Ранний отказ в CI, а не «молча» сломанный деплой |
| Единые лейблы **`app.kubernetes.io/*`** | Селекторы, логи, HPA, стандарты команды |
| Ресурсы **requests/limits** из values | Разные профили по средам |
| Не генерировать пустые блоки | Всегда **`if`** вокруг optional Ingress/PVC |
| Ревью шаблонов как кода | Сложные `range` легко ломают отступы YAML |

---

## Дополнительные материалы

- [Chart templates](https://helm.sh/docs/chart_template_guide/)
- [Sprig functions](https://masterminds.github.io/sprig/)
- [Helm — whitespace control](https://helm.sh/docs/chart_template_guide/control_structures/#whitespace-control)

