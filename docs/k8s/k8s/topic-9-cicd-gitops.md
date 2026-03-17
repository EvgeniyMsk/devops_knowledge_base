# 9. CI/CD и GitOps в Kubernetes

Отдельные страницы: [Helm](../helm/helm.md), [ArgoCD](../../gitops/argocd/argocd.md).

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **CI (Continuous Integration)** | Автоматическая сборка, тесты и публикация артефактов при коммите; в контексте K8s — сборка образа, сканирование, push в registry. |
| **CD (Continuous Delivery/Deployment)** | Доставка или развёртывание в окружение после успешного CI; в K8s — обновление манифестов/чартов в кластере. |
| **GitOps** | Подход: состояние кластера описывается в Git; контроллер (Argo CD, Flux) синхронизирует кластер с репозиторием; изменения только через коммиты. |
| **Helm** | Пакетный менеджер для Kubernetes: чарт (Chart) — набор шаблонов манифестов и values; установка/обновление через helm install/upgrade. |
| **Chart (чарт)** | Пакет Helm: директория с Chart.yaml, values.yaml и templates/; параметризуемые манифесты для приложения или компонента. |
| **Release** | Экземпляр установленного чарта в кластере с именем и версией; один чарт может быть установлен несколько раз под разными именами. |
| **Argo CD** | GitOps-контроллер: следит за Git-репозиторием (или Helm chart) и синхронизирует состояние кластера; UI и поддержка нескольких источников. |
| **Flux** | GitOps-контроллер: операторы для синхронизации Git → кластер, Helm releases, образы; работает в кластере без отдельного UI. |
| **Application (Argo CD)** | Объект CRD: описывает источник (Git path, Helm chart) и назначение (cluster, namespace); Argo CD применяет манифесты и отслеживает drift. |
| **Sync (синхронизация)** | Приведение состояния кластера к описанному в Git; ручной или автоматический (auto-sync); при расхождении — apply или replace. |

---

## Обзор: Git → CI → Helm → GitOps → Cluster

Типичный поток: **исходный код и манифесты в Git** → **CI** (сборка образа, тесты, сканирование, публикация образа и/или Helm-чарта) → **GitOps-контроллер** (Argo CD или Flux) следит за репозиторием с манифестами/чартами и синхронизирует состояние кластера. Кластер всегда приводится к состоянию, описанному в Git; изменения — только через коммиты и PR. Ниже — компоненты и примеры.

---

## Helm

**Helm** — пакетный менеджер для Kubernetes: чарт (Chart) — набор шаблонов и values; установка/обновление через `helm install` / `helm upgrade`. В CI/CD чарты хранят в репозитории (Chart Museum, OCI registry, Git); GitOps подтягивает чарт из репо и применяет в кластер.

### Структура чарта

```text
myapp/
  Chart.yaml          # Метаданные чарта (name, version, appVersion)
  values.yaml         # Значения по умолчанию
  charts/             # Зависимости (subcharts)
  templates/          # Шаблоны манифестов (*.yaml, _*.yaml — partials, *.tpl — helpers)
  templates/NOTES.txt # Текст после установки
  templates/tests/    # helm test
```

### Chart.yaml (пример с комментариями)

```yaml
# Метаданные чарта; version — семантическое версионирование чарта (для Helm).
apiVersion: v2
name: myapp
description: My Application Helm chart
type: application
version: 1.2.0
appVersion: "1.2.0"
keywords:
  - web
  - api
maintainers:
  - name: Platform Team
    email: platform@example.com
dependencies: []
```

### values.yaml: параметризация

```yaml
# Значения по умолчанию; переопределяются через -f или --set при установке.
# В GitOps (Argo CD) передают values из Application или отдельный values file в Git.
replicaCount: 2
image:
  repository: myregistry.example.com/myapp
  tag: ""   # Обычно задаётся в CI или в values для окружения
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"
```

### Шаблон Deployment (templates/deployment.yaml)

```yaml
# Использование values и встроенных объектов (Release, Chart, Capabilities).
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### Хелперы (templates/_helpers.tpl)

```yaml
# Общие шаблоны для имён и меток; вызываются через include.
{{/*
  fullname: имя релиза или release-name-chartname
*/}}
{{- define "myapp.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "myapp.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Helm hooks (отложенные действия)

```yaml
# Hooks выполняются в определённые фазы (pre-install, post-upgrade и т.д.).
# Аннотация helm.sh/hook-weight задаёт порядок при нескольких хуках одной фазы.
apiVersion: batch/v1
kind: Job
metadata:
  name: myapp-migrate
  annotations:
    helm.sh/hook: pre-upgrade,pre-install
    helm.sh/hook-weight: "-5"
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp:{{ .Values.image.tag }}
          command: ["/bin/run-migrations"]
```

---

## GitOps: Argo CD и Flux

**GitOps:** источник истины — Git; контроллер в кластере сравнивает желаемое состояние (манифесты или рендер Helm из Git) с фактическим и применяет изменения (sync). **Argo CD** и **Flux** — два популярных инструмента; оба поддерживают Helm-чарты из Git и OCI.

### Argo CD: Application (Helm из Git)

```yaml
# Application описывает, что синхронизировать: Helm-чарт из репозитория.
# Арго периодически сравнивает желаемое состояние с кластером и при sync применяет манифесты.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://git.example.com/platform/myapp-charts.git
    targetRevision: main
    path: charts/myapp
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

- **syncPolicy.automated:** при расхождении Argo сам выполняет sync; **prune** удаляет ресурсы, удалённые из Git; **selfHeal** откатывает ручные изменения в кластере.
- **valueFiles** и **parameters** переопределяют values чарта (например, образ из CI).

### Argo CD: стратегии синхронизации

```yaml
# syncOptions и retry при сбоях; для StatefulSet/баз данных часто отключают auto-sync
# и используют manual sync или sync waves (annotations argocd.argoproj.io/sync-wave).
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

- **PruneLast** — удаление ресурсов в конце sync (безопаснее для зависимостей). **Sync waves** (например, `sync-wave: "-1"` для Namespace, `"0"` для CRD, `"1"` для приложения) задают порядок применения.

### Flux: HelmRelease (кратко)

Во Flux деплой Helm обычно описывается через **HelmRelease**: указание репозитория, чарта, values и интервала обновления. Установка и настройка — через `flux bootstrap` и CRD; принцип тот же — состояние из Git, автоматическая синхронизация.

---

## CI: сборка образа, сканирование, деплой

CI (GitLab CI, Jenkins, GitHub Actions и др.) обычно: **собирает образ** (Dockerfile), **сканирует** его (уязвимости), **пушит** в registry с тегом (например, по коммиту или версии), при необходимости **собирает и пушит Helm-чарт** (или обновляет values в Git с новым тегом образа). GitOps-контроллер подхватывает новое состояние из Git или обновлённый образ через параметр (image.tag).

### GitLab CI: сборка, сканирование, публикация образа (пример)

```yaml
# .gitlab-ci.yml — фрагмент: build, scan, push. Переменные (REGISTRY, CI_REGISTRY_*) задаются в GitLab.
stages:
  - build
  - scan
  - push

variables:
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$IMAGE_TAG -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$IMAGE_TAG
    - docker push $CI_REGISTRY_IMAGE:latest
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request"
      when: never
    - when: on_success

# Сканирование на уязвимости (Trivy); при критичных можно fail pipeline.
scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 0 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$IMAGE_TAG
  allow_failure: true
```

### Передача версии образа в GitOps

Варианты:

1. **CI обновляет Git:** коммит в репо с манифестами/чартами (например, обновление `values-production.yaml` с `image.tag: <новый тег>`). Argo CD/Flux видят изменение и синхронизируют.
2. **CI передаёт образ в Argo CD:** через CLI или API обновить Application (helm.parameters image.tag) и при необходимости выполнить sync. Источник истины для манифестов остаётся Git; только тег образа приходит из CI.
3. **Image Updater (Argo CD):** Argo CD Image Updater может обновлять теги образов в Application по метаданным registry (по тегам или digest).

Пример обновления values в Git из CI (скрипт):

```bash
# Условный пример: подставить новый тег в values и закоммитить (в реальности — через API/токен).
# sed -i "s|tag:.*|tag: $NEW_TAG|" charts/myapp/values-production.yaml
# git add charts/myapp/values-production.yaml && git commit -m "chore: image $NEW_TAG" && git push
```

---

## Best practices и примеры из production

### Helm

- **Версионирование чарта и appVersion:** увеличивать версию при изменении шаблонов/values; в CI использовать осмысленные теги образов (semver или commit SHA).
- **values по окружениям:** отдельные файлы (values-production.yaml, values-staging.yaml) или параметры в Argo CD Application; не хранить секреты в values в Git — использовать External Secrets или Argo CD + sealed secrets.
- **Хуки:** использовать для миграций БД (pre-upgrade); задавать hook-delete-policy, чтобы не засорять кластер; учитывать, что при неудаче хука релиз может остаться в состоянии pending-upgrade.
- **Тесты:** `helm test` для smoke-проверок после установки; в CI можно запускать `helm template` и линтеры (helm lint, kubeconform).

### GitOps

- **Один репозиторий приложений или монорепо с путями по сервисам:** явная структура (например, `apps/<name>/`, `charts/<name>/`) упрощает права доступа и обзоры.
- **Automated sync с осторожностью:** для БД и stateful-сервисов часто отключают selfHeal или используют manual sync; для stateless — automated с prune и selfHeal приемлемы.
- **Sync waves и порядок:** Namespace и CRD — раньше, приложения — позже; при необходимости использовать Argo CD sync waves (annotations).
- **Разделение окружений:** отдельные Application (или директории) для staging/production; production — только из main с обязательным код-ревью.

### CI

- **Сканирование образов** (Trivy, Clair) в pipeline; при критичных уязвимостях — fail или блокировка деплоя.
- **Подписывание образов** (cosign, notary) и политики допуска в кластер (e.g. admission controller) повышают безопасность.
- **Не деплоить из CI напрямую в кластер** (kubectl apply из job) в production — предпочтительно GitOps: CI только пушит образ и при необходимости обновляет Git; деплой делает Argo CD/Flux.

---

## Практика: Git → CI → Helm → Argo CD → Cluster

1. **Репозиторий приложения:** исходный код, Dockerfile, тесты. CI при пуше в main: сборка образа, сканирование, push в registry с тегом (например, `v1.2.3` или `sha-abc123`).
2. **Репозиторий с чартами (или тот же монорепо):** Helm-чарт приложения (Chart.yaml, values.yaml, templates). Для production — values-production.yaml с нужными ресурсами и ingress.
3. **Argo CD Application:** указывает на путь к чарту в Git, destination namespace, valueFiles (в т.ч. values-production.yaml). Образ можно передавать через helm.parameters (image.tag) — обновляется вручную или из CI (скрипт/API).
4. **Синхронизация:** Argo CD периодически сравнивает Git и кластер; при включённом automated sync выполняет apply; при ручном — sync по кнопке или `argocd app sync`.
5. **Проверка:** после sync убедиться, что поды в Running, сервис доступен; при использовании хуков миграций — проверить логи Job.

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Манифесты и образы из Git + registry** | Один источник истины; аудит через Git history. |
| **Helm для параметризации по окружениям** | Один чарт, разные values; меньше дублирования. |
| **CI не деплоит в production напрямую** | Деплой только через GitOps; CI только build + push + обновление Git. |
| **Сканирование образов в CI** | Раннее обнаружение уязвимостей до деплоя. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Секреты в values в Git** | Утечка при доступе к репо. | External Secrets, Sealed Secrets, Vault. |
| **Ручной kubectl apply в production** | Расхождение с Git, нет воспроизводимости. | Все изменения через Git + GitOps. |
| **Один values для всех окружений** | Риск перепутать настройки prod/stage. | Отдельные valueFiles или Application на окружение. |
| **Деплой без тестов образа** | Уязвимости и поломки в production. | Сканирование и тесты в CI перед обновлением образа. |

---

## Дополнительные материалы

- [Helm — документация](https://helm.sh/docs/)
- [Argo CD — документация](https://argo-cd.readthedocs.io/)
- [Flux — документация](https://fluxcd.io/docs/)
- [GitLab CI — Docker](https://docs.gitlab.com/ee/ci/docker/)
- [Trivy — container scanning](https://github.com/aquasecurity/trivy)
- [Helm](../helm/helm.md) — обзор в этой базе знаний
- [ArgoCD](../../gitops/argocd/argocd.md) — обзор в этой базе знаний
