# 9. Практика использования

---
Цель — **собрать сквозную систему** уровня production-лаборатории: **Vault HA на Raft**, кластер **Kubernetes**, **CI/CD**, затем включить **Kubernetes auth**, **динамические учётки БД**, **подачу секретов в Pod**, **auto-unseal**, **audit**, а также зафиксировать **стратегию бэкапов** и **мониторинг**. Ниже — целевая архитектура, чеклист реализации, best practices и короткие ориентиры по командам (детали см. в предыдущих разделах).

---

## Целевая архитектура (end-to-end)

```
                    ┌─────────────────┐
   Developers ─────►│      CI/CD      │──► deploy / migrate (короткий доступ к Vault)
                    └────────┬────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│ Kubernetes                                                      │
│  ┌──────────────┐   Kubernetes Auth    ┌─────────────────────┐  │
│  │  Pod +       │◄────────────────────►│ Vault HA (Raft ×3)  │  │
│  │  Injector/   │   JWT SA             │ auto-unseal (KMS)   │  │
│  │  Agent       │                      │ audit → file/SIEM   │  │
│  └──────┬───────┘                      └──────────┬──────────┘  │
│         │ dynamic DB creds                        │             │
└─────────┼─────────────────────────────────────────┼─────────────┘
          ▼                                         │
   ┌─────────────┐                                  │
   │ PostgreSQL  │◄─────────────────────────────────┘
   └─────────────┘
          ▲
   Prometheus ◄─── telemetry / алерты (узлы Vault)
```

Комментарий: в учебном контуре допустим один кластер K8s и одна БД; в production добавьте **отдельные среды** и сетевые политики.

---

## Что реализовать (чеклист)

| # | Компонент | Критерий готовности |
|---|-----------|----------------------|
| 1 | **Vault HA, Raft** | Три узла, `vault operator raft list-peers`, устойчивый лидер после рестарта follower |
| 2 | **Auto-unseal** | После reboot узла кластер становится unsealed без ручного Shamir (KMS вашего облака) |
| 3 | **Kubernetes auth** | Роль с `bound_service_account_*`; login из Pod успешен, токен с ожидаемыми политиками |
| 4 | **Dynamic DB (PostgreSQL)** | `vault read database/creds/...` выдаёт пользователя; после TTL — отзыв в БД |
| 5 | **Secrets injection в Pod** | Agent Injector или Agent sidecar отдаёт файл/шаблон без секретов в образе |
| 6 | **Audit** | Включён `file` и/или `syslog`; события видны при чтении KV и логине |
| 7 | **Backup** | Расписание snapshot + **тестовое восстановление** на изолированном стенде |
| 8 | **Monitoring** | Prometheus scrapes `/v1/sys/metrics`, есть алерт на `sealed` / отсутствие лидера |

---

## 1. Vault HA и auto-unseal

**Production:**

- `api_addr` / `cluster_addr` с **HTTPS**, корректный **retry_join** (или auto-join в облаке).
- **Auto-unseal** — блок `seal "awskms"` / `gcpckms` / `azurekeyvault` в `vault.hcl` (см. раздел про архитектуру).

Фрагмент напоминания (KMS обобщённо):

```hcl
# Уже в кластере: storage "raft" { ... }
seal "awskms" {
  region     = "eu-central-1"
  kms_key_id = "alias/lab-vault-unseal"
}
```

---

## 2. Kubernetes auth + политика приложения

```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
```

```bash
vault write auth/kubernetes/role/demo-app \
  bound_service_account_names=demo-app \
  bound_service_account_namespaces=default \
  policies=demo-db-read \
  ttl=15m
```

Политика `demo-db-read` должна разрешать **только** `database/creds/demo-app` (и при необходимости узкий KV), без `sys/*`.

---

## 3. Dynamic credentials: PostgreSQL

```bash
vault secrets enable database

vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@postgres.demo.svc:5432/appdb?sslmode=disable" \
  allowed_roles="demo-app" \
  username="vault_admin" \
  password="REDACTED"

vault write database/roles/demo-app \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="5m" \
  max_ttl="1h"
```

Комментарий: в production — **TLS к БД** (`sslmode=require`), отдельный `vault_admin` с минимальными правами.

---

## 4. Подача секретов в Pod (Injector)

Минимальные аннотации (имена путей подставьте свои):

```yaml
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "demo-app"
    vault.hashicorp.com/agent-inject-secret-db: "database/creds/demo-app"
    vault.hashicorp.com/agent-inject-template-db: |
      {{- with secret "database/creds/demo-app" -}}
      DATABASE_URL="postgresql://{{ .Data.username }}:{{ .Data.password }}@postgres.demo.svc:5432/appdb"
      {{- end }}
spec:
  serviceAccountName: demo-app
```

Приложение читает смонтированный файл с `DATABASE_URL` (или парсит JSON — по вашему шаблону).

---

## 5. Audit

```bash
vault audit enable -path=audit-file file \
  file_path=/var/log/vault/audit.log \
  mode=0600
```

Best practice: отдельный том под логи, ротация, сбор в SIEM; следить за заполнением диска.

---

## 6. CI/CD в контуре проекта

Условие лаборатории: пайплайн **не** хранит root token; используйте **OIDC/JWT** или **AppRole** с маскированием вывода.

```yaml
# Фрагмент GitLab CI (идея)
deploy:
  script:
    - export VAULT_TOKEN="$(vault write -field=token auth/jwt/login role=ci-deploy jwt=$CI_JOB_JWT)"
    - vault kv get -mount=secret ci/helm | yq ...
```

Комментарий: роль `ci-deploy` — отдельная политика только на нужные пути и **короткий TTL**.

---

## 7. Стратегия бэкапов

| Элемент | Действие |
|---------|----------|
| **Raft snapshot** | Cron / агент: `vault operator raft snapshot save` → объектное хранилище с шифрованием |
| **DR** | Для лаборатории достаточно snapshot; для enterprise-географии см. replication |
| **Доказательство** | Раз в квартал: restore на песочнице и smoke-тест |

---

## 8. Мониторинг

- В `vault.hcl`: блок `telemetry` с `prometheus_retention_time`.
- Prometheus: `metrics_path: /v1/sys/metrics`, `format=prometheus`, токен только на метрики.
- Обязательные алерты: **sealed**, отсутствие **leader**, рост ошибок **5xx** на login.

---

## Критерии качества выполненной практики

1. Pod без секретов в образе получает **DATABASE_URL** из Vault и успешно подключается к PostgreSQL.
2. При удалении Pod **новый** комплект dynamic credentials выдаётся заново.
3. После перезагрузки ноды Vault **входит в unseal** без ручного ввода ключей.
4. В audit видна цепочка: **login** → **read** database creds.
5. Есть **скрипт** или документ с расписанием snapshot и ссылкой на тест restore.

---

## Дополнительные материалы

- Сводка по разделам: [**1. База**](topic-1-basis.md) → [**5. Интеграции**](topic-5-integrations.md) → [**7. Эксплуатация**](topic-7-operations.md).

