# 2. Продвинутая логика pipeline

---
Эта тема учит управлять сложными сценариями в GitLab CI/CD: условный запуск job’ов (`rules`), управление графом выполнения через `needs` (DAG), параллельные задачи через `parallel:matrix`, динамические пайплайны (включение шаблонов/файлов и child pipelines), а также разбиение конфигурации на несколько файлов через `include`.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **rules** | Условия запуска job’а: `if`, `changes`, `exists`, плюс `when`. Позволяют точно контролировать, где и когда выполнять шаг. |
| **Dynamic pipeline** | Пайплайн, который “складывается” из условий: включаемые шаблоны, разные ветки исполнения, child pipelines. |
| **DAG (Directed Acyclic Graph)** | Граф зависимостей без циклов. В GitLab выражается через `needs`, когда job’и могут стартовать раньше окончания stage. |
| **parallel / matrix** | Параллельные job’ы, параметризованные матрицей переменных. Удобно для сборки/тестов в разных средах. |
| **Child pipeline** | Вложенный пайплайн, запускаемый через `trigger:`. Может иметь собственные stages и jobs. |
| **Parent pipeline** | “Верхний” пайплайн, который запускает child pipeline и ожидает его по стратегии. |
| **include** | Механизм подключения дополнительных конфигураций из локальных/удалённых файлов или шаблонов GitLab. |

---

## `rules`: if, changes, exists (production-подход)

### Определение

`rules` оцениваются до запуска job’а. Рекомендуется использовать `rules` вместо `only/except`, потому что логика становится читабельнее и предсказуемее.

### Пример: условный deploy только на main + с учётом изменений

```yaml
deploy:
  stage: deploy
  script:
    - ./deploy.sh
  rules:
    # 1) Разрешаем деплой только при коммитах в main
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_success
    # 2) Дополнительно: деплой только если затронуты файлы приложения
    - changes:
        - app/**/*
        - Dockerfile
      when: on_success
    # 3) Всё остальное — не запускать
    - when: never
```

Комментарий (production):

- держите правила **жёсткими** (последнее `when: never`) — так job не “случится” в неожиданном сценарии;
- минимизируйте количество условий: проще сопровождать.

---

## `needs`: DAG вместо “ждать весь stage”

### Идея

В классическом варианте job’и ждут завершения всех job’ов в предыдущем stage. `needs` позволяет запускать job’и раньше, как только готовы конкретные зависимости.

### Мини-проект (lint → build → test параллельно с подготовкой)

```yaml
stages:
  - build
  - test

build-image:
  stage: build
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
  artifacts:
    paths:
      - build-manifest.json

unit-tests:
  stage: test
  needs:
    - job: build-image
      artifacts: true
  script:
    - pytest -q
```

Best practice:

- используйте `needs` для **ускорения** и уменьшения длительности pipeline;
- не добавляйте `needs` “на всякий случай” — это усложняет граф.

---

## Параллельные job’ы: `parallel:matrix`

### Зачем

Когда нужно выполнить одну и ту же задачу для нескольких комбинаций переменных (версии языка, среды, регионы).

### Пример: тесты на нескольких версиях Python

```yaml
test:
  stage: test
  image: python:3.12-slim
  parallel:
    matrix:
      - PY_VERSION: ["3.10", "3.11", "3.12"]
  script:
    - python --version
    - pip install -r requirements.txt
    - pytest -q
  rules:
    - if: '$CI_PIPELINE_SOURCE != "schedule"'
```

Production best practice:

- не делайте матрицу слишком широкой (это быстро раздувает время и стоимость);
- делайте переменные матрицы детерминированными и используйте кэш зависимостей (cache) при необходимости.

---

## Dynamic pipelines: `include` и child pipelines

### Include: разбиение на файлы

Используйте `include`, чтобы вынести повторяющиеся определения (например, общие job’ы или шаблоны сборки).

```yaml
include:
  - local: '/.gitlab/ci/common.yml'
  - local: '/.gitlab/ci/backend.yml'
  - template: Security/SAST.gitlab-ci.yml

stages:
  - lint
  - build
  - test
  - deploy
```

Best practice:

- включайте только то, что действительно нужно этому проекту;
- держите структуру `/.gitlab/ci/` аккуратной (меньше “магии”).

---

### Child pipeline: триггер микросервиса

Сценарий: parent pipeline собирает “монорепо”, а каждый микросервис запускает свои job’ы как child.

```yaml
stages:
  - build
  - deploy

deploy-microservice-a:
  stage: deploy
  trigger:
    include:
      - local: '/services/a/.gitlab-ci.yml'
    strategy: depend   # parent ждёт результат child
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_success
    - when: never
```

Production best practice:

- стратегия `depend` полезна, когда deploy microservice должен считаться “частью” общего результата;
- всегда добавляйте `rules` у trigger job’а, иначе child pipeline может стартовать неожиданно.

---

## Production чеклист для сложных pipeline’ов

- Используйте `rules` с финальным `when: never`.
- Применяйте `needs` для DAG и ускорения.
- Масштабируйте с `parallel:matrix`, но держите матрицу разумной.
- Разделяйте конфиг через `include`, не превращая `.gitlab-ci.yml` в “простыню”.
- Для микросервисов используйте child pipelines с `trigger` и понятной стратегией ожидания.

---

## Дополнительные материалы

- [GitLab CI/CD — rules](https://docs.gitlab.com/ee/ci/yaml/#rules)
- [GitLab CI/CD — needs](https://docs.gitlab.com/ee/ci/yaml/#needs)
- [GitLab CI/CD — parallel:matrix](https://docs.gitlab.com/ee/ci/yaml/#parallelmatrix)
- [GitLab CI/CD — include](https://docs.gitlab.com/ee/ci/yaml/includes.html)
- [GitLab CI/CD — child pipelines](https://docs.gitlab.com/ee/ci/pipelines/child_parent.html)
