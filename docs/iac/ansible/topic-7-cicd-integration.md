# 7. CI/CD и интеграция: GitLab CI, Jenkins, lint и тесты ролей

---
Как встроить Ansible в процессы команды: запускать lint и проверки в CI, делать syntax check, организовать безопасный deploy, тестировать роли через Molecule и подход Test‑driven Ansible. В разделе — небольшие примеры кода, комментарии и production best practices.

---

## Интеграция с GitLab CI / Jenkins

Принцип одинаковый: pipeline должен делать **проверки** до деплоя и иметь управляемый способ запуска deploy (ручной для production или по правилам).

Типовой пайплайн:

- `lint` — стиль и best practices (ansible-lint, yamllint)
- `syntax-check` — проверка синтаксиса playbook и инвентаря
- `deploy` — запуск playbook на окружение (с vault, ограничениями, tags)

---

## Проверки

### ansible-lint

**ansible-lint** ловит ошибки идемпотентности, сомнительные паттерны (shell без необходимости), проблемы с именованием и best practices.

```bash
ansible-lint
```

Production best practices:

- фиксировать версию ansible-lint в CI
- настроить правила/исключения, но не выключать lint «в ноль»

### Molecule

**Molecule** — фреймворк для тестирования ролей: поднимает тестовую среду (Docker/Podman/VM), применяет роль, проверяет идемпотентность и assertions.

```bash
# Инициализация тестов для роли
cd roles/nginx
molecule init scenario -r nginx -d docker

# Прогон тестов
molecule test
```

### Test-driven Ansible (идея)

Для каждой роли:

- написать минимальные tests/assertions (например, service running, порт открыт)
- прогонять molecule в CI на PR
- держать роль идемпотентной (повторный прогон не меняет состояние)

---

## Практика: pipeline (lint → syntax check → deploy)

### Пример `.gitlab-ci.yml`

```yaml
stages: [lint, check, deploy]

lint:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install ansible ansible-lint yamllint
    - yamllint .
    - ansible-lint

syntax-check:
  stage: check
  image: python:3.12-slim
  script:
    - pip install ansible
    - ansible-playbook -i inventories/dev/hosts.ini site.yml --syntax-check

deploy-dev:
  stage: deploy
  image: python:3.12-slim
  rules:
    - if: $CI_COMMIT_BRANCH == \"main\"
  script:
    - pip install ansible
    # Пример: деплой только конфигов
    - ansible-playbook -i inventories/dev/hosts.ini site.yml --tags config
```

Комментарии:

- Для production обычно делают `when: manual` и отдельные правила доступа.
- Секреты (SSH ключи, vault password) должны приходить из CI secret store, а не из репозитория.

### Пример Jenkins (концептуально)

В Jenkins pipeline этапы такие же: `sh 'ansible-lint'`, `sh 'ansible-playbook --syntax-check'`, затем deploy‑шаг с параметрами окружения.

---

## Практика: Molecule тест для роли

Мини‑пример сценария `molecule/default/molecule.yml` (идея):

```yaml
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: instance
    image: \"geerlingguy/docker-ubuntu2204-ansible:latest\"
provisioner:
  name: ansible
verifier:
  name: ansible
```

Пример проверки идемпотентности обычно делается самим `molecule test` (он применяет роль повторно и проверяет, что изменений нет).

---

## Production best practices

- **Разделять deploy и проверки**: lint/check всегда на PR, deploy — по правилам
- **Pinned versions**: фиксировать версии Ansible, ansible-lint, коллекций/ролей
- **Секреты**: SSH ключи и vault password только из secret storage CI, доступ по окружениям
- **Безопасный production deploy**:
  - `--check`/`--diff` на этапе ревью (где возможно)
  - `serial` и healthchecks для rolling обновлений
  - ручной gate (manual approval) для production
- **Артефакты**: не сохранять логи с секретами; использовать `no_log: true` в задачах с секретами

