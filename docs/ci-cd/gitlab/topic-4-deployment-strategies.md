# 4. Deployment стратегии

---
Эта тема учит **безопасно деплоить** через GitLab CI/CD: пользоваться **environments**, делать **review apps**, использовать **manual jobs** и **protected branches**, применять **blue/green** и **canary** подходы, а также выстраивать **rollback** стратегии.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Environment** | Логическое окружение (production, staging, review/feature-*). GitLab связывает деплой с environment и даёт history/rollback возможности через UI/API (зависит от настроек). |
| **Review app** | Автоматический деплой для Merge Request (обычно отдельный namespace/URL), который живёт пока существует MR. |
| **Manual job** | Job, который запускается вручную (кнопкой), часто для approve-процесса перед production. |
| **Protected branch** | Ветка, к которой доступ ограничен ролями/правилами. Часто используется, чтобы deploy выполнялся только на защищённые ветки. |
| **Blue/Green deployment** | Стратегия: одновременно поддерживаются два окружения (blue и green). Переключение трафика на “зелёное” после успешной проверки. |
| **Canary deployment** | Стратегия: новая версия получает небольшой процент трафика или ограниченное количество реплик, после чего доля увеличивается при успешных метриках. |
| **Rollback** | Возврат к предыдущей рабочей версии после ошибок/ухудшения метрик. |
| **Release** (в контексте деплоя) | Пакет “что именно деплоили”: образ/тег + набор параметров конфигурации. |

---

## Environments: как связать деплой с историей

Production best practice:

- деплой должен быть привязан к `environment`, чтобы было понятно “что сейчас живёт”;
- использовать отдельные environment для staging/production и динамические для MR.

Мини-пример:

```yaml
deploy_staging:
  stage: deploy
  script:
    - kubectl -n staging apply -f k8s/
  environment:
    name: staging
    url: https://staging.example.com
```

---

## Review apps для Merge Request

### Зачем

- быстрый feedback на реальность изменений;
- снижение стоимости ошибок до production.

### Мини-подход: динамическое имя + stop job

```yaml
review_app:
  stage: deploy
  image: bitnami/kubectl:1.30
  script:
    - echo "Deploy for MR: $CI_MERGE_REQUEST_IID"
    - kubectl config set-context --current --namespace="review-$CI_MERGE_REQUEST_IID"
    - kubectl apply -f k8s/
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://review.example.com/$CI_MERGE_REQUEST_IID
    on_stop: stop_review_app
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: on_success

stop_review_app:
  stage: deploy
  image: bitnami/kubectl:1.30
  script:
    - kubectl delete -f k8s/ --ignore-not-found
    - kubectl delete namespace "review-$CI_MERGE_REQUEST_IID" --ignore-not-found
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    action: stop
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
```

Комментарий:

- `on_stop` нужен, чтобы review ресурсы автоматически чистились;
- namespace review лучше делать отдельным, чтобы не конфликтовать между MR.

---

## Manual jobs и approve перед production

Production best practice:

- production deploy — только с manual approval;
- запуск — только на protected branch (или при наличии protected variables).

Пример:

```yaml
deploy_production:
  stage: deploy
  image: bitnami/kubectl:1.30
  script:
    - kubectl -n production rollout restart deploy/myapp
  environment:
    name: production
  rules:
    # Запуск только когда пайплайн на protected ветке (обычно main).
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
```

---

## Protected branches: защита от “случайного production”

Основная идея: если ветка защищена, вы снижаете риск, что несанкционированные изменения попадут в production.

В production pipeline:

- deploy_production должен “разрешаться” только на `main`/`release-*`, и только если вы выполняете правила GitLab Protected Branches.

Дополнительно:

- используйте protected/ masked variables для секретов (и отмечайте их protected).

---

## Blue/Green и переключение

### Концептуальная реализация в Kubernetes

В Kubernetes обычно делают два Deployment/две группы pod’ов и переключают Service selector или Ingress backend.

Production best practice:

- прогонять smoke тесты на новом (green) окружении;
- переключение делать атомарно (одним шагом), чтобы избежать “смешения” версий.

Мини-пример “голого” переключения через переменную:

```yaml
deploy_blue:
  stage: deploy
  script:
    - echo "Deploy to BLUE slot"
    - ./deploy.sh --slot blue --image "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  environment: { name: production/blue }

deploy_green:
  stage: deploy
  script:
    - echo "Deploy to GREEN slot"
    - ./deploy.sh --slot green --image "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  environment: { name: production/green }

switch_traffic:
  stage: deploy
  script:
    # атомарно переключаем Service selector (пример, зависит от вашей схемы)
    - kubectl -n production patch svc myapp -p '{"spec":{"selector":{"version":"green"}}}'
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## Canary deployment (через переменные)

Production best practice:

- canary делайте через параметр (weight/replica/namespace), а не через “ручные правки”;
- всегда собирайте метрики перед расширением доли.

Пример: canary переключатель через переменную `CANARY_WEIGHT`.

```yaml
deploy_canary:
  stage: deploy
  image: bitnami/kubectl:1.30
  variables:
    CANARY_WEIGHT: "10" # 10% на новую версию
  script:
    # Helm/Kustomize — пример: передаём значения в chart
    - helm upgrade --install myapp ./chart \
        --namespace production \
        --set image.tag="$CI_COMMIT_SHA" \
        --set canary.weight="$CANARY_WEIGHT"
  environment: { name: production }
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## Rollback стратегии: что делать при провале

Есть два подхода:

- **именно rollback релиза** (helm rollback / восстановление предыдущей версии);
- **переключение трафика обратно** (blue/green — вернуться на blue).

Production best practice:

- у rollback должен быть чёткий триггер: таймаут, алерты, smoke test fail;
- rollback должен быть “коротким” и предсказуемым.

Мини-пример “rollback” через Helm:

```bash
# Пример логики (идея): при неуспехе деплоя откатить релиз.
helm upgrade --install myapp ./chart --namespace production
if [ $? -ne 0 ]; then
  # В реальности используйте конкретную предыдущую revision.
  helm rollback myapp --namespace production
  exit 1
fi
```

---

## Production чеклист deployment’а

- Каждый деплой привязан к `environment`.
- Review apps создаются на MR и очищаются через `on_stop`.
- Production deploy — только `manual` + protected branch.
- Canaries/blue-green — только параметризованные процессы.
- Rollback заранее продуман и протестирован на staging.
- Все секреты и доступ ограничены protected variables.

---

## Дополнительные материалы

- [GitLab Environments](https://docs.gitlab.com/ee/ci/environments/)
- [Review Apps](https://docs.gitlab.com/ee/ci/review_apps/)
- [Manual actions](https://docs.gitlab.com/ee/ci/yaml/index.html#whenmanual)
- [Protected branches](https://docs.gitlab.com/ee/user/project/protected_branches.html)
