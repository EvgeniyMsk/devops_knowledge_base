# 5. Интеграции

---
Цель — связать Vault с **Kubernetes**, **CI/CD** и **приложениями** так, чтобы секреты не лежали в образах и репозиториях, а выдавались **по политике**, с **коротким временем жизни** и понятным аудитом. Ниже — паттерны: **Agent Injector**, **CSI**, пайплайны **GitLab/GitHub**, **Vault Agent** с шаблонами.

---

## Kubernetes: три типичных пути

| Подход | Идея | Когда уместно в production |
|--------|------|----------------------------|
| **Vault Agent Injector** | Mutating admission webhook вешает **init/sidecar** Vault Agent на Pod; секреты оказываются в **файлах** (или env через шаблон). | Самый распространённый путь с Helm-чартом Vault; удобно для legacy-приложений «читаю файл». |
| **Sidecar** (тот же Agent) | Явный sidecar в манифесте вместо автоматических аннотаций — больше контроля, больше YAML. | Строгие политики GitOps, кастомные объёмы/права. |
| **CSI Secrets Store** | CSI-драйвер монтирует секреты; **Secrets Store CSI + Vault provider** синхронизирует с `Secret` K8s при необходимости. | Нужна интеграция с **нативными** `Secret`/`volume`, единый UX с другими провайдерами CSI. |

Best practices (общие):

- **Kubernetes auth** в Vault с узкими `bound_service_account_*`;
- TLS до Vault, доверие к **CA** внутри кластера;
- не монтировать секреты в **world-readable** пути; `defaultMode` 0400 где уместно;
- мониторинг **401/403** и задержек на старте Pod.

---

## Vault Agent Injector: идея и аннотации

Инжектор подставляет контейнер **vault-agent** (и часто init), который **логинится** по Kubernetes JWT и **рендерит** секреты в volume.

Комментарий: точный набор аннотаций зависит от версии чарта/инжектора — сверяйтесь с [документацией](https://developer.hashicorp.com/vault/docs/platform/k8s/injector) для вашей версии.

### Минимальный пример фрагмента Pod

Webhook сам добавит init/sidecar и тома; в контейнере приложения секрет обычно доступен под `/vault/secrets/...` (путь уточните по версии инжектора).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payments-api
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "payments"
    # Секрет из KV v2 → файл (имя и mount задаёт инжектор)
    vault.hashicorp.com/agent-inject-secret-api-key: "secret/data/payments/api"
    vault.hashicorp.com/agent-inject-template-api-key: |
      {{- with secret "secret/data/payments/api" -}}
      {{ .Data.data.key }}
      {{- end }}
spec:
  serviceAccountName: payments
  containers:
    - name: app
      image: myregistry/payments:1.4.0
      # Приложение читает смонтированный агентом файл, путь — из документации/аннотаций
```

Best practice: для шаблонов со **множеством** полей используйте **один** файл-конфиг (JSON/YAML), чтобы приложение читало один mount, а не десятки файлов.

---

## Sidecar vs CSI — как выбрать

| Критерий | Agent Injector / Sidecar | CSI (Vault provider) |
|----------|---------------------------|----------------------|
| Перезапуск при ротации | Agent умеет **рендерить заново** по настройкам | Зависит от sync/Pod events |
| Зависимости | Под webhook + образ agent | DaemonSet CSI + provider |
| Нативный `Secret` в namespace | Нужен доп. шаг или скрипт | Часто проще выстроить sync → `Kubernetes Secret` |
| Blast radius компрометации node | Как у любого hostPath/emptyDir — думайте про права | Аналогично; проверяйте sync в etcd |

---

## CI/CD: GitLab CI и GitHub Actions

Задачи пайплайна: **имперсонация** (OIDC/JWT), **короткий token**, **динамические** креды к БД/облаку вместо статики.

### Общие production-правила

- **Не** хранить root token в variables; **OIDC → Vault** или **AppRole** с секретом в protected/masked variables.
- Минимальный TTL токена на job; **одна роль** на тип деплоя (dev/prod).
- Логи CI **маскировать** вывод `vault kv get`; не печатать полные ответы.
- Dynamic credentials: креды живут только на время шага deploy/migrate.

### GitLab CI (упрощённо): логин и чтение

```yaml
deploy:
  image: hashicorp/vault:1.15
  variables:
    VAULT_ADDR: "https://vault.example:8200"
  script:
    # Пример: токен из защищённой CI variable (лучше заменить на JWT/OIDC по гайду Vault+GitLab)
    - export VAULT_TOKEN="$VAULT_CI_TOKEN"
    - vault kv get -mount=secret -format=json ci/apps/deploy | jq -r '.data.data.ssh_key' > ssh_key.pem
    - chmod 600 ssh_key.pem
    # дальше деплой; файл не коммитить
  after_script:
    - rm -f ssh_key.pem
```

Комментарий: для **GitLab OIDC** настраивается `jwt`/`oidc` auth в Vault и роль с `bound_claims` — предпочтительнее долгоживущего `VAULT_CI_TOKEN`.

### GitHub Actions: официальный action

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: hashicorp/vault-action@v3
        with:
          url: https://vault.example:8200
          method: jwt
          role: github-actions-deploy
          secrets: |
            secret/data/ci/github kv_token | KV_TOKEN
      - name: Use secret
        run: echo "Token length ${#KV_TOKEN}"
        # не выводить значение в лог
```

Best practice: роль `github-actions-deploy` в Vault должна быть привязана к **`sub`/`repository`** claim’ам GitHub.

---

## Приложения: Vault Agent, шаблоны, env vs файлы

### Режимы Agent (ориентир)

- **Авто-аутентификация** + **template** → файл(ы) на диске;
- Приложение **старует после** готовности файла (init container или entrypoint `until [ -f ... ]`).

### Фрагмент `agent.hcl` (пример)

```hcl
pid_file = "/tmp/agent.pid"
vault {
  address = "https://vault.example:8200"
}
auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config = { role = "payments" }
  }
  sink "file" {
    config = { path = "/home/vault/.vault-token" }
  }
}
template {
  destination = "/vault/secrets/config.env"
  contents = <<EOT
{{- with secret "secret/data/payments/config" -}}
DATABASE_URL="{{ .Data.data.url }}"
API_KEY="{{ .Data.data.api_key }}"
{{- end }}
EOT
}
```

### Переменные окружения vs файлы

| Способ | Плюсы | Риски в production |
|--------|--------|---------------------|
| **Файл** | Проще убрать из `/proc/*/environ`, удобно для 12-factor с `env_file` | Права на файл, путь mount |
| **Env** | Быстро подключить к существующему приложению | Утекает в дампы env, некоторые логгеры |
| **Сокет / volume только для приложения** | Жёсткая изоляция | Сложнее в отладке |

Best practice: для чувствительных значений отдавать **ф файл** + `defaultMode: 0400`; env — только если приложение не умеет иначе, и без логирования старта с полным окружением.

---

## Практика: что должно получиться

1. **Kubernetes**: Pod с `serviceAccountName` и политикой Vault получает секрет **без ручного `kubectl create secret`** из репозитория с открытым текстом.
2. **CI**: пайплайн получает доступ **на время job** и не оставляет долгоживущих токенов в артефактах.

---

## Сводный checklist

| # | Практика |
|---|----------|
| 1 | Kubernetes auth: узкие `bound_*`, отдельный SA на сервис |
| 2 | Не класть долгоживущие token’ы Vault в Git variables без ротации |
| 3 | OIDC/JWT для GitLab/GitHub предпочтительнее статического token |
| 4 | Динамические креды для deploy/migrate к БД |
| 5 | Шаблоны Agent: один файл конфигурации, минимум копий секрета в Pod |

---

## Дополнительные материалы

- [Vault Agent Injector](https://developer.hashicorp.com/vault/docs/platform/k8s/injector)
- [Vault Agent](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent)
- [CSI provider for Vault](https://developer.hashicorp.com/vault/docs/platform/k8s/csi)
- [vault-action (GitHub)](https://github.com/hashicorp/vault-action)

