# 6. Оптимизация и performance

---
Цель темы — сделать CI/CD в GitLab **быстрым и дешёвым**: меньше лишних job’ов, правильные `cache`, минимизация `artifacts`, параллелизация, Docker layer caching и запуск только нужных шагов через `rules:changes` и `needs`.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Pipeline** | Набор job’ов, выполняемых по событиям. |
| **Job** | Отдельная задача в `.gitlab-ci.yml`. |
| **Cache** | Кэш между job’ами/пайплайнами для ускорения повторяющихся шагов. |
| **Artifacts** | Результаты job’а, которые передаются в следующие шаги. |
| **Layer caching (Docker)** | Кэширование слоёв образа при повторной сборке (уменьшает время сборки). |
| **Parallelization** | Запуск нескольких job’ов или параллельных матриц одновременно. |
| **rules:changes** | Запуск job’а только если изменились определённые файлы/пути. |
| **DAG через needs** | Граф зависимостей без ожидания конца stage: job стартует сразу после готовности `needs`. |

---

## Что чаще всего замедляет pipeline

- лишние job’ы (запускаются на каждый коммит, даже если не затронута нужная часть)
- большие `artifacts`, которые таскаются везде
- кэш без стратегии (слишком общий cache = частые промахи)
- Docker build без эффективного caching (не тот `Dockerfile`, нет layer reuse)
- ожидание всего stage вместо DAG (`needs`)

---

## 1) Запуск только нужных jobs: `rules:changes`

Production best practice: сначала “срежьте” pipeline до минимума, а потом оптимизируйте кэш/артефакты.

Пример: линтер только если изменён код/зависимости.

```yaml
lint:
  stage: test
  image: python:3.12-slim
  script:
    - ruff check .
  rules:
    - changes:
        - "services/**/**"
        - "requirements*.txt"
        - "pyproject.toml"
      when: on_success
    - when: never
```

Комментарий:

- `when: never` в конце — страховка от “случайного запуска”.
- Пути должны совпадать с реальной структурой репозитория.

---

## 2) Cache стратегии: быстрый и предсказуемый

Кэш ускоряет, но только если:

- ключ кэша детерминирован и меняется при изменении зависимостей;
- кэш не содержит “мусор”, который ломает сборку.

Production подход (пример с pip):

```yaml
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

build:
  stage: build
  image: python:3.12-slim
  cache:
    key:
      # Меняем кэш, когда меняются зависимости.
      files:
        - "requirements.txt"
    paths:
      - .cache/pip
  script:
    - pip install -r requirements.txt
    - pytest -q
```

Best practice:

- используйте `cache:key:files` (или эквивалент) вместо “одного ключа на всё”.
- не кэшируйте директории, которые зависят от среды, если они не гарантируют воспроизводимость.

---

## 3) Минимизация `artifacts`

`artifacts` полезны, но стоят времени/трафика.

Rule of thumb:

- храните **только то, что нужно дальше** (например, test reports, build outputs)
- ограничивайте `expire_in`, чтобы артефакты не копились бесконечно
- не делайте “универсальные artifacts”, которые каждый job таскает

Пример:

```yaml
test:
  stage: test
  script:
    - pytest -q
  artifacts:
    when: always
    expire_in: 2 weeks
    reports:
      junit: report.xml
```

---

## 4) Параллелизация и matrix (но без взрыва стоимости)

Параллельность ускоряет, но увеличивает общий расход.

Production best practice:

- матрицу делайте по “реальным измерениям” (окружение/версия/регион), которые нужны вам по тест-плану
- ставьте ограничение на общее число job’ов в матрице

Пример: тесты на разных версиях.

```yaml
test:
  stage: test
  parallel:
    matrix:
      - PY_VERSION: ["3.10", "3.11", "3.12"]
  script:
    - python --version
    - pip install -r requirements.txt
    - pytest -q
```

---

## 5) Docker layer caching

Если вы собираете Docker образы каждый раз с нуля — pipeline будет медленным.

Паттерн:

- использовать buildkit/buildx или канонический процесс для кеша
- сохранять/использовать cache layers через registry cache (зависит от инструмента)

Минимальный production-принцип:

- цель — чтобы следующий build использовал слои прошлого build при совпадении инструкций/файлов.

Пример (идея через buildx cache; точная команда зависит от вашего runner’а):

```bash
docker buildx build \
  --cache-from=type=registry,ref="$CI_REGISTRY_IMAGE:buildcache" \
  --cache-to=type=registry,ref="$CI_REGISTRY_IMAGE:buildcache",mode=max \
  -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" \
  --push .
```

---

## 6) Используйте `needs` вместо “ожидать весь stage”

DAG позволяет стартовать тесты сразу после build, а не после завершения всего stage.

```yaml
build:
  stage: build
  script:
    - ./build.sh

test:
  stage: test
  needs:
    - job: build
      artifacts: true
  script:
    - ./test.sh
```

---

## Production чеклист: ускорить pipeline минимум в 2 раза

1) сначала режем ненужные job’ы через `rules:changes`
2) потом добавляем адекватные `cache` для зависимостей
3) затем уменьшаем `artifacts`: только essentials + короткий `expire_in`
4) включаем `needs` для DAG
5) добавляем параллельность там, где она реально ускоряет (matrix)
6) оптимизируем Docker layer caching под ваш runner/инструмент

---

## Дополнительные материалы

- [GitLab CI/CD — cache](https://docs.gitlab.com/ee/ci/caching/)
- [GitLab CI/CD — artifacts](https://docs.gitlab.com/ee/ci/pipelines/job_artifacts.html)
- [GitLab CI/CD — rules: changes](https://docs.gitlab.com/ee/ci/yaml/#changes)
- [GitLab CI/CD — needs](https://docs.gitlab.com/ee/ci/yaml/#needs)
