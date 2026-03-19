# 11. Helm и GitOps

---
**Helm** — инструмент **упаковки и шаблонизации**; **GitOps** — модель **доставки**: желаемое состояние в **Git**, в кластере — **контроллер согласования** (pull). Сами по себе **`helm upgrade` из CI** не равны GitOps, если Git не считается **единственным источником истины** для того, что реально должно быть в Kubernetes. Ниже — различие подходов, роль Helm в **Argo CD** и **Flux**, минимальные примеры манифестов и production-замечания.

---

## Почему «только Helm» не GitOps

| Признак GitOps (упрощённо) | Типичный деплой «Helm из CI» |
|----------------------------|------------------------------|
| **Декларация в Git** — эталон | В Git может быть chart, но «истина» — последний успешный пайплайн |
| **Контроллер** постоянно **подтягивает** желаемое состояние | Агент **push** в API по событию |
| **Дрифт** относительно Git **лечится** (self-heal) или фиксируется алертом | Ручные `kubectl` правки могут жить до следующего деплоя |

Вывод: Helm **совместим** с GitOps — как **движок рендеринга** или как **формат поставки** chart’а — но GitOps — это **процесс и инструменты**, а не один бинарник `helm`.

---

## Helm + Argo CD

Argo CD берёт **chart** из Git (как каталог) или из **Helm repository**, выполняет **`helm template`** (логически) и применяет манифесты, отслеживая **sync** с веткой/тегом.

### Фрагмент `Application` (ориентир)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops.git
    targetRevision: main
    path: charts/my-app
    helm:
      valueFiles:
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: "1.4.2"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
```

Best practices:

- **отдельный** GitOps-репозиторий или строгая структура монорепо;
- **не** хранить сырые секреты в `values-prod.yaml` — Sealed/SOPS/External Secrets;
- версии chart’а фиксировать **`targetRevision`** (тег/коммит), не «плавующий» main без политики.

Углубление по Argo CD и Helm: [**Работа с Helm и Kustomize**](../../gitops/argocd/topic-7-helm-kustomize.md).

---

## Helm + Flux

**Flux** использует CR **`HelmRelease`**: источник — **HelmRepository** / **OCI** или chart из **GitRepository**. Контроллер периодически **сверяет** установленный релиз с объявлением в Git.

### Фрагмент `HelmRelease` (ориентир)

Версию **`apiVersion`** (`v2`, `v2beta2` и т.д.) возьмите из **`kubectl api-resources`** на своём кластере Flux.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: apps
spec:
  interval: 10m
  chart:
    spec:
      chart: my-app
      version: "1.4.x"
      sourceRef:
        kind: HelmRepository
        name: my-charts
        namespace: flux-system
  values:
    replicaCount: 3
    image:
      tag: "1.4.2"
```

Best practices:

- **`interval`** и **версия chart’а** согласовать с окном релизов;
- крупные values вынести в **`ConfigMap`**/`Secret` refs Flux или отдельный YAML в Git, если политика позволяет;
- наблюдаемость за **Ready** статусом `HelmRelease`.

---

## Как Helm «внутри» GitOps

| Роль Helm | Что делает контроллер |
|-----------|------------------------|
| **Chart в Git** | Клонирует репозиторий, подставляет values, применяет |
| **Chart из repo / OCI** | Скачивает версию, ставит/обновляет **Helm release** в кластере |
| **Параметры** | Values из CR или файлов Git → тот же движок шаблонов |

В обоих случаях **полезно** продолжать **lint/template** в CI для быстрой обратной связи до merge.

---

## Production: смешивание push-Helm и GitOps

| Риск | Митигация |
|------|-----------|
| Один и тот же релиз **то** CI, **то** Argo/Flux | Единый **владелец** пути деплоя; политика «только GitOps» или «только pipeline» |
| **Дрифт** от ручных правок | Self-heal в Argo / запрет на ручной edit критичных полей |
| Разные **values** в CI и в Git | Один набор prod-values, второй только для локальных тестов |

---

## Краткий чеклист

| # | Вопрос |
|---|--------|
| 1 | Где **каноническое** желаемое состояние — в Git для оператора? |
| 2 | Кто **создаёт** Helm release — Flux `HelmRelease` или Argo `Application`? |
| 3 | Секреты и **версии** chart’а/образа аудируются? |
| 4 | Есть ли алерт на **OutOfSync** / **не дефолтный Ready** у HelmRelease? |

---

## Дополнительные материалы

- [Argo CD — Helm](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Flux — Helm releases](https://fluxcd.io/flux/components/helm/helmreleases/)
- [OpenGitOps principles](https://opengitops.dev/)

