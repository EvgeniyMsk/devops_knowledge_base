# 3. Архитектура платформы (templates и pipeline-as-a-platform)

---
Цель темы — мыслить как системный инженер и строить CI/CD в GitLab как **платформу**: единые переиспользуемые шаблоны (templates), управляемые версии, понятные контракты параметров и предсказуемые правила запуска.

В конце вы должны уметь:

- выбирать monorepo vs multirepo подход для pipeline’ов,
- делать reusable templates (внутри проекта и между проектами),
- версионировать шаблоны так, чтобы production-деплой был воспроизводим,
- рассматривать GitOps-подходы при интеграции с кластерами/репозиториями.

---

## Определения

| Термин | Определение |
|--------|-------------|
| **Monorepo** | Один репозиторий для множества сервисов/пакетов. |
| **Multirepo** | Отдельные репозитории на каждый сервис/компонент. |
| **Reusable templates** | Шаблоны CI, которые используются в нескольких проектах или в нескольких частях одного проекта. |
| **Pipeline as a platform** | Идея: pipeline даёт стандартизированные шаги и контракты (что нужно, как запускается, какие артефакты создаёт), как внутренняя “платформа”. |
| **Template versioning** | Версионирование шаблонов CI (обычно через tags/commits), чтобы production исполнял конкретную версию логики. |
| **Include** | Механизм подключить конфигурацию из файла/проекта/шаблона. |
| **Child pipeline** | Пайплайн, который запускает parent пайплайн. Удобно для масштабирования конфигурации по сервисам. |
| **Trigger** | Запуск pipeline в другом репозитории/проекте. |
| **GitOps** | Подход, при котором desired-state хранится в Git, а деплой делается контроллерами (например, Argo CD/Flux). |

---

## Monorepo vs Multirepo: как влияет на CI

### Monorepo

Плюсы:

- проще поддерживать единый набор шаблонов и стандартов,
- легче делать “один pipeline для всех”, используя `rules:changes`.

Минусы:

- pipeline может стать “слишком общим”, если не ограничивать запуск по путям,
- растёт цена на проверку зависимостей.

Best practice:

- использовать `rules:changes`, чтобы запускать только релевантные job’ы,
- вынести общую логику в templates и параметризовать.

### Multirepo

Плюсы:

- независимые release-потоки,
- меньше “лишней” работы в pipeline.

Минусы:

- труднее обеспечить единый стандарт и одинаковые шаги,
- шаблоны CI приходится шарить между проектами.

Best practice:

- шарить шаблоны через include из отдельного “platform templates” проекта,
- версионировать include через tags/commits.

---

## Reusable templates: внутри проекта и между проектами

### Внутри одного репозитория

Если есть повторяющиеся шаги сборки/тестов:

```yaml
# .gitlab-ci.yml

stages: [build, test]

.build-template:
  stage: build
  image: docker:27
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .

build:
  extends: .build-template
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

test:
  stage: test
  image: python:3.12-slim
  script:
    - pytest -q
```

Комментарий:

- шаблон задаёт “контракт” шага сборки,
- job’ы в разных местах расширяют его через `extends`.

### Между проектами: include с версионированием

Best practice:

- всегда пиньте `ref:` к **tag** (или commit SHA), чтобы не ловить неожиданные изменения шаблонов.

Пример:

```yaml
# .gitlab-ci.yml в сервисе

include:
  - project: "platform/ci-templates"
    file: "/build/docker-build.yml"
    ref: "v1.2.3"

variables:
  IMAGE_NAME: "orders-api"
  DOCKERFILE_PATH: "services/orders/Dockerfile"
```

В шаблоне вы читаете входные переменные и формируете jobs.

---

## Pipeline as a platform: контракт входов/выходов

Системный подход к шаблонам:

- **Input контракт**: какие переменные должны быть заданы (`IMAGE_NAME`, `DOCKERFILE_PATH`, `DEPLOY_ENV`).
- **Output контракт**: какие артефакты/имена образов появляются (например, `$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA`).
- **Semantics**: что job делает и чего не делает (например, template build **только строит**, deploy делает отдельный шаг).

Production правило:

- шаблон должен быть “узким”: один смысл = один шаблон.
- документация шаблона обязана быть в репозитории templates (README рядом с файлами).

---

## Версионирование шаблонов: как избежать “сломали production”

Рекомендуемый процесс:

1. В отдельном проекте templates выпускаете tag `vX.Y.Z`.
2. Сервисы в `.gitlab-ci.yml` подключают templates строго по `ref: vX.Y.Z`.
3. Обновление ref делается через PR, проходит код-ревью и отдельную проверку.

Пример стратегии:

```text
platform/ci-templates v1.2.3 -> stable
platform/ci-templates v1.3.0 -> next (в тестовых проектах)
```

---

## GitOps подходы: как не дублировать ответственность

Если у вас GitOps контролируется Argo CD или Flux:

- CI/CD pipeline обычно **не делает “kubectl apply” напрямую в production**,
- вместо этого pipeline обновляет Git репозиторий с манифестами (image tag / values),
- GitOps контроллер сам приводит кластер к desired-state.

Best practice:

- CI делает commit/PR в конфиг-репозитории,
- контроллер (Argo/Flux) следит за дрейфом и выполняет синхронизацию.

Мини-пример (идея): pipeline формирует новый tag и обновляет манифесты в Git.

```bash
# Идея: обновить image tag в values и запушить PR
NEW_TAG="$CI_COMMIT_SHA"
yq -i ".image.tag = \"$NEW_TAG\"" "deploy/values.yaml"
git commit -am "chore: bump image to $NEW_TAG"
git push origin HEAD:gitops-branch
```

Комментарий:

- конкретная реализация зависит от вашего GitOps инструмента,
- важно ограничивать права токена (минимальные права для записи в нужную ветку/папку).

---

## Дополнительные материалы

- [GitLab CI/CD — include](https://docs.gitlab.com/ee/ci/yaml/includes.html)
- [GitLab CI/CD — extends](https://docs.gitlab.com/ee/ci/yaml/#extends)
- [GitLab CI/CD — Child Pipelines](https://docs.gitlab.com/ee/ci/pipelines/parent_child_pipelines.html)
