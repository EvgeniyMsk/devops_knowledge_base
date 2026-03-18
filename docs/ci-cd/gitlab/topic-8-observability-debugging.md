# 8. Observability и отладка

---
Цель темы — понимать, что происходит внутри CI/CD в GitLab: какие job’ы когда стартуют, почему падают, где “зависают”, и как диагностировать flaky/race conditions. В production это помогает сократить MTTR и избежать “слепых” пайплайнов.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Job logs** | Логи выполнения job’а (stdout/stderr), которые позволяют понять причину успеха/ошибки. |
| **Pipeline metrics** | Численные показатели пайплайна: длительности, доля падений, частота повторов, очереди раннеров. |
| **Flaky job** | Job, который иногда проходит, иногда падает, без изменения кода (зависит от таймингов/окружения). |
| **Race condition** | Сбой из-за конкуренции: два потока/процесса делают операции в неправильном порядке из‑за скорости/параллельности. |
| **Triage (разбор инцидента)** | Быстрая классификация причины: инфраструктура vs тесты vs код vs внешние зависимости. |
| **Artifact** | Результат job’а, который можно скачать/использовать для диагностики (логи, отчёты). |

---

## Pipeline “как система наблюдаемости”

В production пайплайн стоит рассматривать как систему:

- где есть входы (commit/MR/schedule),
- где есть зависимые сервисы (registry, DB, external API),
- где есть узкие места (раннеры, кэш, сеть, docker build),
- и где нужны сигналы (метрики/логи/уведомления).

Best practice: у каждого job’а должен быть “диагностический след”:

- достаточно логов,
- понятные error messages,
- (опционально) artifacts с report’ами/сырыми логами.

---

## Как смотреть, что происходит: job logs и timing

### Практика: включить полезные логи

```yaml
test:
  stage: test
  script:
    # Быстро находим “тормоза”
    - date
    - echo "Runner: $CI_RUNNER_ID"
    - echo "Job started: $CI_JOB_STARTED_AT"
    - ./test.sh
  after_script:
    # Снимайте дополнительную диагностику, если job упал.
    - 'if [ "$CI_JOB_STATUS" != "success" ]; then ls -la; fi'
  rules:
    - when: on_success
```

Комментарий:

- `after_script` — хороший способ собрать контекст для расследования.
- не забывайте про секреты: не логируйте значения masked/protected переменных.

### Практика: собрать artifacts для диагностики

```yaml
test:
  stage: test
  script:
    - pytest -q --junitxml=report.xml
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      junit: report.xml
    paths:
      - logs/
```

---

## Диагностика flaky jobs

### Симптомы

- job падает редко, но не стабильно;
- падает на MR/CI чаще, чем локально;
- причина часто “в таймингах”: ожидание сервиса, порт, readiness.

### Подход: найти “недетерминизм”

1) Зафиксируйте окружение: версии образов, параметры тестов, переменные.
2) Проверьте параллельность: не делите один и тот же ресурс между job’ами без изоляции.
3) Добавьте задержки/timeout там, где есть внешняя зависимость, но аккуратно.

Best practice:

- вместо “sleep 10” используйте readiness wait (poll по health endpoint/портам) с timeout.

Мини‑пример “wait for service”:

```bash
#!/usr/bin/env bash
set -euo pipefail

host="${1:-db}"
port="${2:-5432}"
timeout_sec="${3:-60}"

start="$(date +%s)"
while ! nc -z "$host" "$port"; do
  if [ $(( $(date +%s) - start )) -ge "$timeout_sec" ]; then
    echo "Timeout waiting for $host:$port" >&2
    exit 1
  fi
  sleep 2
done

echo "Service is reachable: $host:$port"
```

---

## Race conditions из-за параллельности

Частые причины:

- общий workspace/директории (если раннер переиспользует filesystem),
- общий namespace/окружение для нескольких job’ов,
- конфликты при создании ресурсов (один job “перекрывает” другой).

Best practices:

- изолируйте ресурсы по `CI_PIPELINE_ID`, `CI_JOB_ID` или MR ID (namespace/названия тестовых ресурсов),
- используйте `needs`/`dependencies`, чтобы граф был предсказуемым,
- избегайте side effects между job’ами.

Мини‑пример: уникальный namespace для integration tests:

```yaml
integration:
  stage: test
  variables:
    TEST_NS: "test-$CI_MERGE_REQUEST_IID"
  script:
    - kubectl create namespace "$TEST_NS" || true
    - kubectl -n "$TEST_NS" apply -f k8s/
    - kubectl -n "$TEST_NS" rollout status deploy/myapp --timeout=120s
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

---

## Failure patterns: как классно реагировать на падения

### Классификация ошибок (быстро)

- инфраструктура: раннер/сеть/registry/время ожидания,
- flaky tests: тайминги/ready,
- код/регрессия: стабильно воспроизводится,
- секреты/права: masked/protected, permissions.

Best practice:

- для инфраструктурных сбоев допустим retry,
- для кодовых ошибок retry только ухудшает картину.

Пример ограниченного retry:

```yaml
build:
  stage: build
  script:
    - ./build.sh
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
  rules:
    - when: on_success
```

---

## Уведомления: Slack/email для MTTR

Production best practice:

- уведомлять не “каждый раз”, а только на MR/главных ветках и при критичных ошибках;
- добавлять в уведомление ссылку на pipeline + короткий “reason”.

Концептуальный пример: notify на failure (через скрипт/CLI).

```yaml
notify_on_failure:
  stage: deploy
  script:
    - ./notify.sh --status "$CI_JOB_STATUS" --url "$CI_PIPELINE_URL"
  when: on_failure
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

Комментарий:

- реализация `notify.sh` зависит от вашей интеграции (Slack webhook, email, etc).

---

## Production чеклист

- Каждый job логирует контекст (версии образов, параметры, тайминги).
- Добавлены artifacts для диагностики упавших job’ов.
- Flaky устранён через readiness wait, изоляцию ресурсов и детерминизм.
- Retry ограничен типами “инфраструктурных” сбоев.
- Есть уведомления на main/production ветках.

---

## Дополнительные материалы

- [GitLab CI/CD — Job logs](https://docs.gitlab.com/ee/ci/jobs/job_logs.html)
- [GitLab CI/CD — Retry](https://docs.gitlab.com/ee/ci/yaml/index.html#retry)
