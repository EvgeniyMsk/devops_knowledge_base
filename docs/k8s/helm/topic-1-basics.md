# 1. Основы Helm

---
Helm — **пакетный менеджер** для Kubernetes: вы описываете приложение как **chart**, устанавливаете и обновляете его как **release**, версионируете и переиспользуете через **репозитории**. В **Helm v3** нет **Tiller** — всё выполняется через **Kubernetes API** от имени вашего `kubeconfig`. Ниже — термины, структура чарта, базовые команды и production-замечания.

---

## Термины

| Термин | Смысл |
|--------|--------|
| **Chart** | Пакет: шаблоны манифестов + значения по умолчанию + метаданные |
| **Release** | Установленный **экземпляр** chart’а в кластере с именем и версией релиза |
| **Repository** | HTTP-сервер с **index.yaml** и упакованными `.tgz` chart’ами |
| **Values** | Параметры, которыми подставляются шаблоны (`values.yaml` и `-f` / `--set`) |

---

## Зачем Helm в production

- **Повторяемость**: один chart → dev/stage/prod с разными `values`.
- **Откат**: `helm history` / `helm rollback`.
- **Документация как код**: chart в Git, review как у приложения.
- **Зависимости**: `dependencies` в `Chart.yaml` для составных продуктов.

Best practice: не считать Helm заменой **GitOps** — chart часто **рендерится** в CI/ArgoCD, а Helm ставит только там, где это принято в команде.

---

## Архитектура Helm v3 (без Tiller)

```
helm install/upgrade
       │
       ▼
  Helm-клиент (CLI)
       │
       ├── рендер шаблонов (Go templates + Sprig)
       └── kubectl apply / server-side apply через Kubernetes API
```

**Tiller** был в Helm 2 и выполнял привилегированную роль в кластере; **v3** это убрал — меньше attack surface и проще RBAC.

---

## Структура чарта

```
my-app/
├── Chart.yaml          # метаданные чарта и зависимости
├── values.yaml         # значения по умолчанию
├── templates/          # шаблоны .yaml (Deployment, Service, …)
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl    # общие шаблоны и функции
└── charts/             # вложенные chart’ы (опционально)
```

### `Chart.yaml` (пример)

```yaml
apiVersion: v2
name: my-app
description: Учебный chart приложения
type: application
version: 0.1.0
appVersion: "1.0.0"
```

Production: **`version`** — версия **chart** (semver); **`appVersion`** — ориентир на версию образа приложения; в CI привязывайте bump к релизному процессу.

### `values.yaml` (фрагмент)

```yaml
replicaCount: 2

image:
  repository: ghcr.io/myorg/my-app
  tag: ""  # часто переопределяют в CI: --set image.tag=$GIT_SHA
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

Best practice: **секреты** не хранить в открытом `values.yaml` в Git — **External Secrets**, **Sealed Secrets**, **SOPS** или inject из CI.

---

## Практика: базовые команды

Создать каркас chart’а:

```bash
helm create my-app
```

Установить релиз из локальной папки:

```bash
helm install my-app ./my-app --namespace apps --create-namespace
```

Обновить релиз после правок chart’а или values:

```bash
helm upgrade my-app ./my-app -n apps
```

Удалить релиз и связанные ресурсы (если не переопределён `helm.sh/resource-policy`):

```bash
helm uninstall my-app -n apps
```

Production-ориентиры:

- **`helm upgrade --install`** — идемпотентный пайплайн (если нет — install, если есть — upgrade).
- **`--atomic`** — откат при неуспешном upgrade (с Helm 3, проверьте версию и поведение с hook’ами).
- **`--timeout`** — явный таймаут ожидания готовности.

```bash
helm upgrade --install my-app ./my-app -n apps \
  --atomic \
  --timeout 5m \
  -f values-prod.yaml
```

---

## `helm template` (отладка без кластера)

Рендер манифестов **в stdout** — обязательный инструмент перед `install` и в **CI**.

```bash
helm template my-release ./my-app --debug
```

С переопределением values:

```bash
helm template my-release ./my-app \
  -f values.yaml \
  -f values-prod.yaml \
  --set replicaCount=3
```

Флаги **`--debug`** показывают, какие файлы шаблонов участвовали и помогают ловить ошибки **YAML**/шаблонов.

Best practice: в **merge request** гонять `helm template` + `kubeconform` / `kubectl apply --dry-run=server` по политике команды.

---

## Краткий production checklist

| # | Практика |
|---|----------|
| 1 | Chart и app версии по **semver**, теги образов **не** `latest` в prod |
| 2 | Отдельные **`values-*.yaml`** по средам, минимум дублирования |
| 3 | Секреты вне Git, либо зашифрованы |
| 4 | `helm template` / dry-run в CI |
| 5 | Документировать **обязательные** values в `README` chart’а |

---

## Дополнительные материалы

- [Helm — документация](https://helm.sh/docs/)
- [Charts — best practices](https://helm.sh/docs/chart_best_practices/)
- [Введение в Helm 3](https://helm.sh/docs/intro/)

