# 5. Безопасность CI/CD

---
Цель темы — **не допустить утечек** и **не скомпрометировать** инфраструктуру: правильно работать с секретами, ограничивать доступ к production, изолировать runner’ы и включать security scanning (SAST/DAST/dependency scanning/secret detection).

---

## Определения и сущности

| Термин | Определение |
|--------|-------------|
| **Secrets (секреты)** | Пароли, токены, ключи и другие данные, которые не должны попадать в репозиторий или логи. |
| **CI/CD Variables** | Переменные, заданные в GitLab UI/API: могут быть masked и protected. |
| **Masked variable** | Переменная, которую GitLab скрывает в логах (подменяет значения). |
| **Protected variable** | Переменная доступна только на protected ветках/тегах (защита от несанкционированного запуска). |
| **Runner** | Исполнитель job’ов, который запускает `.gitlab-ci.yml` в вашей инфраструктуре. |
| **Shared runner** | Общий runner, который обслуживает много проектов (уровень доверия ниже). |
| **Specific runner** | Выделенный runner для конкретных проектов/команды (уровень доверия выше). |
| **Dependency scanning** | Поиск уязвимостей в зависимостях (npm/pip/maven и т.д.). |
| **SAST** | Static Application Security Testing: анализ исходного кода без запуска приложения. |
| **DAST** | Dynamic Application Security Testing: анализ “снаружи” как чёрный ящик (при необходимости развёртывания тестовой копии). |
| **Secret detection** | Обнаружение секретов (паролей/токенов) в репозитории/артефактах. |
| **Allow failure** | Настройка, когда job при обнаружении проблем не валит весь пайплайн (используется в переходный период). |

---

## Secrets management: как хранить и использовать секреты

### Best practice: секреты только через variables / Vault

- Не хранить секреты в репозитории.
- Не выводить секреты в stdout/stderr.
- Делать secret доступным только там, где нужно (least privilege).

### Masked / protected переменные

Production подход:

1) переменная должна быть `masked`,
2) переменная должна быть `protected`,
3) job, который использует секрет, должен запускаться только на protected ветке.

Пример правила запуска деплоя:

```yaml
deploy_production:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
  script:
    # В логах не должно быть значения SECRET_* переменных.
    - echo "Deploying..."
    - ./deploy.sh
```

Комментарий:

- даже если вы “случайно” сделаете `echo "$SECRET"`, masked переменная должна скрыть значение.
- protected переменная гарантирует доступ только из защищённых условий GitLab.

---

## Пример: подключение секретов из Vault (через `secrets`)

GitLab может подставлять секреты в job из внешнего хранилища (в зависимости от настройки интеграции).

Идея:

```yaml
deploy_production:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
  secrets:
    DB_PASSWORD:
      vault: production/data/app/db_password
      file: false
  script:
    - ./deploy.sh --db-password "$DB_PASSWORD"
```

Best practice:

- передавать секрет только в процесс, который реально нуждается (а не в “каждый job”).
- запрещать вывод значения в логи.
- ротировать секреты (и закладывать пере-подхват в процессе деплоя).

---

## Security scanning: dependency scanning, SAST, Secret detection, DAST

### Встроенные шаблоны GitLab

Production best practice:

- подключать security templates через `include`,
- запускать на MR и main,
- постепенно ужесточать (allow_failure → hard-fail по мере стабилизации).

Пример (ядро):

```yaml
include:
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

stages: [build, test, security, deploy]

dependency_scanning:
  stage: security
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

sast:
  stage: security
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

secret_detection:
  stage: security
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

Комментарий:

- имена job’ов зависят от шаблона GitLab (они обычно совпадают с доками), но принцип тот же: stage + rules + постепенное ужесточение.

### DAST (когда включать)

DAST тяжелее и требует окружения. Production подход:

- запускать DAST только по расписанию или для MR с пометкой,
- отдавать результаты в отчёт, а деплой не валить сразу (на первом этапе).

---

## Runner security: изоляция доверия

### Shared runner vs specific runner

Production подход:

- **проверяемые** job’ы (lint/test/build) можно запускать на shared runners, если нет секретов и нет привилегий;
- job’ы с секретами и production deploy должны быть только на выделенных/защищённых runner’ах.

### Как ограничить доступ в CI

1) Используйте `tags:` чтобы job работал на нужных runner’ах:

```yaml
deploy_production:
  stage: deploy
  tags: ["k8s-prod"]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
  script:
    - ./deploy.sh
```

2) Не выдавайте job’ам привилегии без необходимости:

- избегайте `privileged: true`,
- не запускайте “опасные” контейнеры на runner’ах, которые имеют доступ к секретам.

3) Минимизируйте то, что попадает на runner:

- очищайте workspace,
- не сохраняйте секреты в artifacts.

---

## Production чеклист по безопасности CI/CD

- Секреты только через masked/protected variables или интеграцию с Vault.
- Production deploy запускается только на protected ветках и только через manual approve (или другой процесс).
- Job’ы с секретами выполняются на specific/protected runner’ах (по tags).
- Включены dependency scanning + SAST + Secret detection как минимум на MR и main.
- DAST подключается позже и запускается по более “дорогим” сценариям (schedule/флаги).
- Постепенный переход: allow_failure на старте → hard-fail после стабилизации.

---

## Дополнительные материалы

- [GitLab Security templates](https://docs.gitlab.com/ee/user/application_security/index.html)
- [GitLab protected variables](https://docs.gitlab.com/ee/ci/variables/#protected-variables)
- [Masked variables](https://docs.gitlab.com/ee/ci/variables/#masked-variables)
