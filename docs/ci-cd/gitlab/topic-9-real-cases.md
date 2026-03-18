# 9. Реальные кейсы (обязательно)

---
Эта тема — “практика под требования Middle+”: вы увидите, как GitLab CI/CD проектируется под типичные production‑условия. Внутри — монорепа с большим числом сервисов, миграция с Jenkins, pipeline на 100+ jobs и измеримая оптимизация времени сборки (30 → 10 минут).

---

## Термины и ожидания

| Термин | Определение |
|--------|-------------|
| **Monorepo** | Репозиторий с множеством сервисов/пакетов в одном Git‑project. |
| **Multiplatform pipeline** | Pipeline, который запускает разные типы шагов (lint/test/build/deploy) для разных частей монорепы. |
| **Drift** | Расхождение между тем, что описано в Git, и тем, что реально развернуто (часто актуально при GitOps). |
| **MTTR** | Среднее время восстановления после инцидента (в CI/CD — время от падения до исправления/перезапуска). |
| **Job budget** | Идея “у каждого job’а есть стоимость”: время/стоимость runner’а/нагрузка на сервисы. |

---

## Кейc 1: монорепа с 20 сервисами

### Типичная проблема

- pipeline “летает”, но делает лишнюю работу на каждый коммит
- сложно понять, какие job’ы имеют отношение к изменениям
- растёт стоимость runner’ов и очередь

### Production‑подход: “контракт изменений” через `rules:changes`

Идея: для каждого сервиса запускать job’ы только если изменились файлы этого сервиса.

Пример (скелет):

```yaml
# .gitlab-ci.yml

stages: [lint, build, test]

lint_base:
  image: python:3.12-slim
  script:
    - ruff check .

build_base:
  image: docker:27
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .

lint_service_a:
  extends: lint_base
  stage: lint
  rules:
    - changes:
        - services/service-a/**/*
      when: on_success
    - when: never
  script:
    - ruff check services/service-a

lint_service_b:
  extends: lint_base
  stage: lint
  rules:
    - changes:
        - services/service-b/**/*
      when: on_success
    - when: never
  script:
    - ruff check services/service-b

build_service_a:
  extends: build_base
  stage: build
  rules:
    - changes:
        - services/service-a/**/*
        - Dockerfile.services-a
      when: on_success
    - when: never
```

Комментарий:

- `when: never` — страховка, чтобы job не запускался “случайно”.
- для монорепы обязательно снижайте количество job’ов, которые выполняются на каждом коммите.

### Best practice: выносить логику в templates + include

Когда сервисов много, .gitlab-ci.yml превращается в кашу. Решение:

- общий template build/lint
- сервисы подключают шаблон через `include` + variables

---

## Кейс 2: миграция с Jenkins на GitLab CI/CD

### Типичная проблема миграции

- “перенесли пайплайн 1:1”, но он стал дольше и нестабильнее
- секреты/права не перенесли правильно
- потеряли артефакты, тест‑репорты и отчёты

### Production‑план миграции (пошагово)

1. **Mapping**: stages/jobs Jenkins → stages/jobs GitLab (и фиксируете ожидания: что должно быть в artifacts, что считается “успехом”).
2. **Runner стратегия**: выбираете shared vs specific runner (особенно для docker‑build и секретов).
3. **Секреты**: переносите в masked/protected variables или vault integration.
4. **Гарантия артефактов**: test reports/manifest build должны быть доступны следующим job’ам.
5. **Параллельный запуск**: сначала гоняете GitLab pipeline параллельно с Jenkins на ограниченном наборе веток.

Мини‑пример “сквозного” job’а деплоя:

```yaml
deploy_production:
  stage: deploy
  image: bitnami/kubectl:1.30
  environment: production
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
  script:
    - kubectl -n production apply -f k8s/
    - kubectl -n production rollout status deploy/myapp --timeout=180s
```

Комментарий:

- manual на production + protected ветка снижает риск “случайного деплоя”.
- rollout status делает деплой проверяемым и предсказуемым.

---

## Кейс 3: pipeline на 100+ jobs

### Типичная проблема

- вы переплачиваете за “каждый job на каждый коммит”
- пайплайн становится нечитаемым
- быстрые job’ы ждут медленных из-за stage-барьеров

### Production‑подход: DAG через `needs` + “стрижка” по `rules`

Пример: build завершился — можно запускать тесты, не дожидаясь конца stage.

```yaml
build:
  stage: build
  script:
    - ./build.sh
  artifacts:
    paths: [dist/]

test:
  stage: test
  needs:
    - job: build
      artifacts: true
  script:
    - ./test.sh
```

Best practices:

- используйте `needs` только там, где реально ускоряет (не усложняйте граф “ради графа”)
- выкидывайте job’ы из pipeline через `rules:changes/exists` и `when: never`

---

## Кейс 4: оптимизация времени сборки 30 → 10 минут

### Метрика успеха

- определите, что именно сокращаем: lint/test/build/deploy
- зафиксируйте baseline и цель (например, build с 25 минут до 8–10)

### Production‑техники

1) **Правильный cache** (ключ менять только при изменении зависимостей).

```yaml
build:
  image: python:3.12-slim
  cache:
    key:
      files: ["requirements.txt", "pyproject.toml"]
    paths:
      - .cache/pip
  script:
    - pip install -r requirements.txt
    - python -m build
```

2) **Минимизация artifacts** (не таскайте “лишнее” на каждый job).

3) **Снижение количества запускаемых job’ов** через `rules:changes`.

4) **Docker layer caching / buildkit** (если используете docker build).

Идея через buildx cache:

```bash
docker buildx build \
  --cache-from=type=registry,ref="$CI_REGISTRY_IMAGE:buildcache" \
  --cache-to=type=registry,ref="$CI_REGISTRY_IMAGE:buildcache",mode=max \
  -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" \
  --push .
```

Комментарий:

- оптимизация “быстро” обычно достигается комбинацией: cache + rules + needs.

---

## Production‑чеклист для “реальных кейсов”

- pipeline запускается только когда нужно (`rules:changes`, `when: never`)
- есть DAG ускорения (`needs`) там, где это оправдано
- артефакты минимальные и нужные
- кэш детерминирован по файлам зависимостей
- production deploy — только manual + protected ветка/окружение
- есть диагностический след: job logs + artifacts для расследования

---

## Дополнительные материалы

- [GitLab CI/CD — rules](https://docs.gitlab.com/ee/ci/yaml/#rules)
- [GitLab CI/CD — needs](https://docs.gitlab.com/ee/ci/yaml/#needs)
- [GitLab CI/CD — caching](https://docs.gitlab.com/ee/ci/caching/)
