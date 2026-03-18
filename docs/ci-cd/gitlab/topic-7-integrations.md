# 7. Интеграции

---
Эта тема показывает, как CI/CD в GitLab становится частью платформы: деплой в Kubernetes, запуск Terraform и Ansible из пайплайна, работа с GitLab Environments, и то, как относиться к Auto DevOps (понимать, но не полагаться).

В фокусе production-практики: изоляция infra и app pipeline, корректная работа с секретами, state и access, предсказуемые деплои.

---

## Определения и сущности

| Термин | Определение |
|--------|-------------|
| **Kubernetes manifest** | YAML/JSON описание ресурсов кластера (Deployment, Service, Ingress, Secret и т.п.). |
| **kubeconfig** | Конфигурация доступа kubectl/клиентам к Kubernetes API (сертификаты/токены/кластер). |
| **GitLab Environment** | Объект GitLab для привязки деплоя к логическому окружению (staging, production, review/*), хранения history. |
| **Review app** | Динамический environment для Merge Request, обычно с отдельным URL. |
| **Terraform apply** | Команда Terraform, которая применяет план изменений инфраструктуры (создаёт/обновляет/удаляет ресурсы). |
| **Terraform state** | Файл/хранилище с текущим состоянием инфраструктуры, от которого зависит планирование следующего apply. |
| **Remote state backend** | Удалённое хранилище state (например, S3, GCS, Terraform Cloud) для команды и lock’ов. |
| **Ansible playbook** | YAML-файл с набором tasks для настройки/изменения системы (серверов/VM). |
| **Inventory** | Описание хостов (static/dynamic) для Ansible. |
| **Auto DevOps** | Автоматическая “из коробки” настройка пайплайнов GitLab для сборки/тестов/деплоя. |
| **Infra pipeline / App pipeline** | Разделение пайплайнов: инфраструктура (Terraform) отдельно от приложения (build/deploy), чтобы управлять рисками и доступами. |

---

## GitLab environments + Kubernetes

Production best practice:

- deploy в Kubernetes всегда привязывайте к `environment` в GitLab;
- для MR используйте `review/*` environments и `on_stop`, чтобы окружения не копились.

Мини-пример: деплой в review namespace (концептуально).

```yaml
deploy_review:
  stage: deploy
  image: bitnami/kubectl:1.30
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://review.example.com/$CI_MERGE_REQUEST_IID
    on_stop: stop_review
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  script:
    - kubectl config use-context "$KUBE_CONTEXT"
    - kubectl -n "review-$CI_MERGE_REQUEST_IID" apply -f k8s/
```

Ключевой момент:

- `KUBE_CONTEXT` (или kubeconfig/sa token) храните в protected variables;
- секреты и токены не должны попадать в логи.

---

## Deploy в Kubernetes через CI: минимально безопасная схема

Production подход:

- не хранить kubeconfig как “чистый файл” в репозитории;
- использовать сервисный аккаунт/токен (Kubernetes) или переменную с ограниченными правами;
- ограничить доступ по namespace и ресурсам (RBAC).

Пример job’а с ограничениям по branch и manual approve:

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

- лучше всего деплоить через `apply`/`upgrade`, а не “перезапусками без контроля”;
- smoke тесты (хотя бы HTTP health-check) желательно добавить перед тем, как считать деплой успешным.

---

## Terraform apply через pipeline (и почему infra pipeline отдельно)

Правило production:

- инфраструктура меняет то, на чём живёт приложение;
- поэтому Terraform pipeline должен быть отделён от app pipeline, а доступ к apply ограничен.

### Terraform: best practices по state

- использовать remote state backend;
- включать lock (чтобы не было параллельных apply);
- выдавать токены только на чтение там, где возможно, и только на apply — для ограниченных ролей.

Мини-пример пайплайна: plan на MR, apply только на main + manual approve.

```yaml
terraform_plan:
  stage: infra-plan
  image: hashicorp/terraform:1.8
  script:
    - terraform init
    - terraform plan -out tfplan
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  artifacts:
    paths:
      - tfplan
    expire_in: 1 week

terraform_apply:
  stage: infra-apply
  image: hashicorp/terraform:1.8
  needs:
    - terraform_plan
  script:
    - terraform init
    - terraform apply -auto-approve tfplan
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
  environment: production
```

Комментарий:

- применяйте plan-artifact, а не делайте `apply` “прямо сейчас” без проверки;
- Terraform secrets (access keys) — только masked/protected.

---

## Ansible через CI: когда это уместно

Ansible часто используют, если:

- есть VM/bare metal, где удобнее конфигурировать через playbooks;
- нужен “bootstrap” окружения или настройка сетей/сертификатов.

Production best practice:

- inventory хранить в репозитории или генерировать из Terraform outputs;
- secrets — через variables/Vault (без вывода в логи);
- включить retry/timeout и “проверяемость” (assert/статусы tasks).

Мини-пример job’а:

```yaml
ansible_deploy:
  stage: deploy
  image: python:3.12-slim
  before_script:
    - pip install ansible-core
  script:
    - ansible-playbook -i inventory/prod deploy.yml --check
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
```

Комментарий:

- в примере используется `--check`; в production обычно отдельный ручной “apply” шаг без check.

---

## Auto DevOps: понимать, но не полагаться

Auto DevOps удобен как старт, но в production он часто:

- не соответствует вашим процессам (security scanning, env separation, специфичные runner’ы);
- плохо описывает кастомные шаги и контракты arтефактов;
- усложняет воспроизводимость.

Best practice:

- используйте Auto DevOps как “референс”, но production pipeline делайте явным и версионируемым через templates/include.

---

## Разделение infra и app pipelines: production-эффект

Почему это важно:

- infra changes могут требовать отдельного approval и иметь иной риск;
- app pipeline должен быть независим от того, что “сегодня” меняли Terraform;
- у команд разные права (infra — узкая группа).

Практическая рекомендация:

- infra pipeline: только Terraform plan/apply;
- app pipeline: build/test и деплой в Kubernetes.

---

## Production чеклист интеграций

- deploy’ы привязаны к `environment` и имеют историю;
- токены/секреты защищены masked/protected и не попадают в логи;
- kubeconfig и права Kubernetes ограничены минимально необходимыми RBAC;
- Terraform использует remote state backend и lock’и;
- Terraform apply делается по `plan` из MR/артефакта;
- Ansible secrets и inventory управляются безопасно;
- Auto DevOps не заменяет явные production pipeline'ы.

---

## Дополнительные материалы

- [GitLab Environments](https://docs.gitlab.com/ee/ci/environments/)
- [GitLab Review Apps](https://docs.gitlab.com/ee/ci/review_apps/)
- [GitLab CI/CD — Secrets](https://docs.gitlab.com/ee/ci/secrets/)
- [Terraform state & backends](https://developer.hashicorp.com/terraform/language/settings/backends)
- [Ansible — Playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks.html)
