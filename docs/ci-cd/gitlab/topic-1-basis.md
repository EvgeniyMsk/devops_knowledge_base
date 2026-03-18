# 1. База: структура `.gitlab-ci.yml`

---
Эта тема закрывает базовые пробелы: как устроен `.gitlab-ci.yml`, что такое **stages/jobs/tags**, как правильно использовать **variables**, **cache** и **artifacts**, почему **rules** лучше чем `only/except`, и как собирать граф зависимостей через `needs` и `dependencies`. Везде — production‑ориентированные комментарии и короткие примеры кода.

---

## Определения (термины GitLab CI/CD)

| Термин | Определение |
|--------|-------------|
| **Pipeline** | Граф/набор job’ов, который запускается по событию (push/MR/schedule). |
| **Stage** | Шаг пайплайна; job’ы внутри stage стартуют параллельно, стадии идут по порядку. |
| **Job** | Единица работы в `.gitlab-ci.yml` (скрипт, зависимости, окружение, артефакты). |
| **Runner** | Исполнитель job’ов (сервер/агент), который выполняет `.gitlab-ci.yml` в вашей инфраструктуре. |
| **Shared runner / Specific runner** | Общий runner (shared) vs выделенный runner под конкретную команду/проект. |
| **Tag** | Метки runner’а. Job может требовать runner по `tags:`. |
| **Variables** | Параметры пайплайна (global/job-level/secret vars). |
| **cache** | Кэш для повторного использования между пайплайнами/задачами (ускоряет сборку). |
| **artifacts** | Результаты job’а для последующих стадий/скачиваний (например, build outputs, test reports). |
| **rules** | Условия запуска job’а (лучший способ, чем `only/except`). |
| **needs** | Зависимости “раньше по графу”: позволяет стартовать job до завершения всей стадии. |
| **dependencies** | Какие artifacts из прошлых job’ов нужно скачать в текущий job. |

---

## Продакшн‑скелет пайплайна: lint → build → test → deploy

Ниже — пример, который обычно хорошо проходит в review и в эксплуатации.

```yaml
# .gitlab-ci.yml

stages:
  - lint
  - build
  - test
  - deploy

variables:
  # В production избегайте hardcode паролей — используйте GitLab CI/CD Variables (masked/protected).
  DOCKER_TLS_CERTDIR: "/certs"

lint:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt
    - ruff check .
  rules:
    # Запускаем на merge requests и на main.
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

build:
  stage: build
  image: docker:27
  services:
    - name: docker:27-dind
      command: ["--mtu=1460"] # пример настройки; подстройте под вашу сеть при необходимости
  variables:
    DOCKER_HOST: "tcp://docker:2375"
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    # Детерминированный тег образа.
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  cache:
    # Кэш для зависимостей (пример).
    key: "pip-cache"
    paths:
      - .cache/pip
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  artifacts:
    when: on_success
    expire_in: 1 week
    paths:
      - build-manifest.json

test:
  stage: test
  image: python:3.12-slim
  needs:
    # Ускорение: тесты стартуют сразу после build, без ожидания всей стадии.
    - job: build
      artifacts: true
  script:
    - pip install -r requirements.txt
    - pytest -q
  artifacts:
    when: always
    expire_in: 2 weeks
    reports:
      junit: report.xml
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy:
  stage: deploy
  image: bitnami/kubectl:1.30
  needs:
    - job: test
      artifacts: false
  script:
    - kubectl apply -f k8s/
    - kubectl -n production rollout status deployment/myapp --timeout=180s
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_success
```

Production‑дисциплина:

- **Детерминированные теги** образов: `:$CI_COMMIT_SHA`.
- **rules** вместо `only/except` дают читабельную логику.
- **needs** ускоряет пайплайн.
- **artifacts** — только то, что реально нужно дальше.

---

## `rules`: управление запуском job’ов

Пример “must-have” шаблона:

```yaml
rules:
  - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    when: on_success
  - if: '$CI_COMMIT_BRANCH == "main"'
  - when: never
```

Best practice:

- добавляйте `when: never` как страховку.

---

## `cache` vs `artifacts`

- `cache` — ускорение повторных запусков.
- `artifacts` — перенос результатов (test reports, build outputs) в последующие job’ы.

Антипаттерн:

- хранить большие артефакты в cache (растёт время и нестабильность).

---

## `needs` и `dependencies`

```yaml
test:
  stage: test
  needs:
    - job: build
      artifacts: true
  dependencies:
    - build
```

Best practice:

- используйте `needs` для скорости,
- используйте `dependencies`/`artifacts: true` для контроля “что скачать”.

---

## Runners и tags

```yaml
build:
  tags:
    - docker
    - linux
```

Best practice:

- договориться о тегаx внутри команды,
- использовать выделенные runner’ы для привилегированных задач.

---

## Variables и секреты

Best practice:

- секреты — только в GitLab UI (masked/protected),
- не выводить их в логи.

Пример job-level переменной:

```yaml
test:
  variables:
    PYTHONWARNINGS: "error"
  script:
    - pytest -q
```

---

## Production checklist

- Разделяйте job’ы по смыслам (lint/test/deploy).
- Начинайте с быстрых проверок (lint) и тестов.
- Минимизируйте артефакты и держите их срок жизни разумным.
- Используйте `rules` и детерминированные теги.
- Держите runner’ы и лимиты под контролем.

---

## Дополнительные материалы

- [GitLab CI/CD — Pipelines](https://docs.gitlab.com/ee/ci/pipelines/)
- [GitLab CI/CD — rules](https://docs.gitlab.com/ee/ci/yaml/#rules)
- [GitLab CI/CD — needs](https://docs.gitlab.com/ee/ci/yaml/#needs)
# 1. База: структура `.gitlab-ci.yml`

---
Эта тема закрывает базовые пробелы: как устроен `.gitlab-ci.yml`, что такое **stages/jobs/tags**, как правильно использовать **variables**, **cache** и **artifacts**, почему **rules** лучше чем `only/except`, и как собирать граф зависимостей через `needs` и `dependencies`. Везде — production‑ориентированные комментарии и короткие примеры кода.

---

## Определения (термины GitLab CI/CD)

| Термин | Определение |
|--------|-------------|
| **Pipeline** | Граф/набор job’ов, который запускается по событию (push/MR/schedule). |
| **Stage** | Шаг пайплайна; job’ы внутри stage стартуют параллельно, стадии идут по порядку. |
| **Job** | Единица работы в `.gitlab-ci.yml` (скрипт, зависимости, окружение, артефакты). |
| **Runner** | Исполнитель job’ов (сервер/агент), который выполняет `.gitlab-ci.yml` в вашей инфраструктуре. |
| **Shared runner / Specific runner** | Общий runner (shared) vs выделенный runner под конкретную команду/проект. |
| **Tag** | Метки runner’а. Job может требовать runner по `tags:`. |
| **Variables** | Параметры пайплайна (global/job-level/secret vars). |
| **cache** | Кэш для повторного использования между пайплайнами/задачами (ускоряет сборку). |
| **artifacts** | Результаты job’а для последующих стадий/скачиваний (например, build outputs, test reports). |
| **rules** | Условия запуска job’а (лучший способ, чем `only/except`). |
| **needs** | Зависимости “раньше по графу”: позволяет стартовать job до завершения всей стадии. |
| **dependencies** | Какие artifacts из прошлых job’ов нужно скачать в текущий job. |

---

## Продакшн‑скелет пайплайна: lint → build → test → deploy

Ниже — пример, который обычно хорошо проходит в review и в эксплуатации.

```yaml
# .gitlab-ci.yml

stages:
  - lint
  - build
  - test
  - deploy

variables:
  # В production избегайте hardcode паролей — используйте GitLab CI/CD Variables (masked/protected).
  DOCKER_TLS_CERTDIR: "/certs"

lint:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt
    - ruff check .
  rules:
    # Запускаем на merge requests и на main.
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

build:
  stage: build
  image: docker:27
  services:
    - name: docker:27-dind
      command: ["--mtu=1460"] # пример настройки; подстройте под вашу сеть при необходимости
  variables:
    # Docker in Docker требует прав; на production обычно настраивают безопаснее (kaniko/buildkit).
    DOCKER_HOST: "tcp://docker:2375"
  before_script:
    # Пример: логин в registry через переменные окружения GitLab.
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    # Используем tag по commit sha — детерминированный build.
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  cache:
    # Кэш для зависимостей (пример). Подходит, если у вас есть локальные директории, которые можно кэшировать.
    key: "pip-cache"
    paths:
      - .cache/pip
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  artifacts:
    # Для build в виде кэша обычно не надо artifacts,
    # но можно сохранять metadata/manifest.
    when: on_success
    expire_in: 1 week
    paths:
      - build-manifest.json

test:
  stage: test
  image: python:3.12-slim
  needs:
    # Production best practice: стартовать тесты как только готов build (не ждать end of stage).
    - job: build
      artifacts: true
  script:
    - pip install -r requirements.txt
    - pytest -q
  artifacts:
    # test reports полезны для CI/CD и анализа качества.
    when: always
    expire_in: 2 weeks
    reports:
      junit: report.xml
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy:
  stage: deploy
  image: bitnami/kubectl:1.30
  needs:
    - job: test
      artifacts: false
  script:
    # Пример: деплой по sha. Конкретика зависит от вашего процесса (Helm/Kustomize/Argo).
    - kubectl apply -f k8s/
    - kubectl -n production rollout status deployment/myapp --timeout=180s
  rules:
    # Обычно deploy только на main и только когда pipeline защитила ветку (protected variables).
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_success
```

Комментарий к production‑дисциплине:

- **Детерминированные теги** образов: `:$CI_COMMIT_SHA`.
- **rules** вместо `only/except` дают читабельную логику.
- **needs** ускоряет пайплайн.
- **artifacts** — только то, что реально нужно дальше (меньше шума и быстрее загрузки).

---

## `rules`: лучший способ управлять запуском job’ов

Основные паттерны:

```yaml
rules:
  # 1) Merge requests
  - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    when: on_success
  # 2) Только main
  - if: '$CI_COMMIT_BRANCH == "main"'
  # 3) Остальное — не запускать
  - when: never
```

Best practice:

- Прописывайте явно `when: never` как “страховку”, чтобы не запускать job случайно.

---

## `cache` vs `artifacts`: не перепутайте

- `cache` — ускорение (повторное использование между job’ами/пайплайнами).
- `artifacts` — перенос результата из job’а в job (например, test reports, build outputs).

Типовой антипаттерн:

- пытаться хранить большие артефакты в cache (получите slow downloads и нестабильность)

---

## `needs` и `dependencies`: граф и где брать данные

Если ваш job зависит от артефактов:

```yaml
test:
  stage: test
  needs:
    - job: build
      artifacts: true
  dependencies:
    # Если вы используете artifacts upload в build, перечисляйте, от кого качать.
    - build
```

Best practice:

- использовать `needs` для ускорения запуска,
- использовать `dependencies` (или `artifacts: true` в `needs`) чтобы контролировать “что именно скачать”.

---

## Runners и tags

Как подключать job к нужному runner’у:

```yaml
build:
  tags:
    - docker
    - linux
```

Production best practice:

- не разбрасывать теги без соглашения,
- использовать специфичные runner’ы для “опасных” задач (dind/privileged),
- документировать, какой runner что делает.

---

## Variables: где хранить секреты

Best practice:

- секреты — только в **GitLab UI** (masked/protected),
- не пушить в репозиторий (и не выводить в лог).

Мини‑пример job-level переменной:

```yaml
test:
  variables:
    PYTHONWARNINGS: "error"
  script:
    - pytest -q
```

---

## Производственный checklist

- Избегайте “сложной магии” в `.gitlab-ci.yml`: один смысл на job.
- Пишите быстрые “lint” и “test” job’ы в начале.
- Теги runner’ов и ресурсы (CPU/RAM) держите под контролем.
- Сводите артефакты к необходимому минимуму.
- Обязательно используйте `rules` и “умные” условия запуска.
- Делайте теги образов детерминированными (sha/tag) и откатывайте через git sha.

---

## Дополнительные материалы

- [GitLab CI/CD — Pipelines](https://docs.gitlab.com/ee/ci/pipelines/)
- [GitLab CI/CD — rules](https://docs.gitlab.com/ee/ci/yaml/#rules)
- [GitLab CI/CD — needs](https://docs.gitlab.com/ee/ci/yaml/#needs)
