# 3. Аутентификация

---
Цель — **подключить Vault к инфраструктуре**: кто и как получает доверенную сессию в Vault без «вечных» root token и без секретов в Git. Ниже — основные **auth methods**, типовые сценарии (CI/CD, Kubernetes, люди через SSO) и короткие примеры команд/API.

---

## Auth methods: обзор

| Метод | Типичный сценарий | Production-комментарий |
|--------|-------------------|-------------------------|
| **Token** | Сервисные аккаунты Vault, break-glass, bootstrap | **Не** основной способ для приложений: короткий TTL, выдача через другой auth. |
| **AppRole** | Машины, CI/CD, VM без нативного identity | **Must know**: role_id + secret_id, ограничения по TTL и политикам. |
| **Kubernetes** | Pod → Vault по ServiceAccount | Роль с `bound_service_account_*`, проверка issuer/CA. |
| **LDAP** | Корпоративные пользователи (каталог) | Группы → политики; чаще в legacy / закрытый периметр. |
| **OIDC (SSO)** | Люди: Okta, Keycloak, Entra ID… | Группы/claims → **identity** + policies; единый вход. |

Правило: **аутентификация** отвечает на «кто вы», **политики** — на «что можно». Один метод может выдавать токен с несколькими политиками и метаданными.

---

## Сценарии

| Сценарий | Рекомендуемый путь | Идея |
|----------|-------------------|------|
| **CI/CD** | **AppRole** (или OIDC workload в пайплайне, если доступен) | Разный role на dev/prod; `secret_id` не хранить в репозитории. |
| **Pod в Kubernetes** | **Kubernetes auth** | Доверие к API K8s + привязка к namespace/SA. |
| **Человек** | **OIDC** (+ MFA на стороне IdP) | Аудит по субъекту IdP, отзыв через IdP. |

---

## Token: напоминание

Токен Vault — это **credential после успешного login** (или ручная выдача оператором). В production:

- минимальный TTL, `renewable` только где осмысленно;
- **не** протаскивать root/service token в переменные образа;
- предпочитать **orphan** и явные политики при автоматизации (см. документацию по `token_type` для вашего auth).

```bash
# Пример: создать токен с политикой (только для операторских сценариев / отладки)
vault token create -policy=read-only-kv -ttl=8h -renewable=true
```

---

## AppRole (обязательно знать)

**role_id** — публичный идентификатор роли; **secret_id** — секрет доставки (ограниченный по TTL/использованиям).

Best practices:

- разные **роли** для сред (dev/stage/prod);
- короткий TTL токена и политики **least privilege**;
- доставлять `secret_id` через секретный канал (Vault Wrap, CI secrets, KMS), **не** в Git;
- при возможности — ограничение по **CIDR** / metadata на роли.

### Включение и роль для CI

```bash
vault auth enable approle

# Роль пайплайна: только нужные policy, ограниченное время жизни токена
vault write auth/approle/role/ci-deploy \
  token_policies="ci-kv-read" \
  token_ttl=15m \
  token_max_ttl=1h \
  bind_secret_id=true \
  secret_id_ttl=60m \
  secret_id_num_uses=10
```

```bash
# Получить role_id (кладётся в CI как non-secret или через переменную окружения по процессу)
vault read auth/approle/role/ci-deploy/role-id

# Выдать secret_id (одноразово — не логировать и не коммитить)
vault write -f auth/approle/role/ci-deploy/secret-id
```

### Логин по AppRole

```bash
export VAULT_ADDR='https://vault.example:8200'

vault write -field=token auth/approle/login \
  role_id="$VAULT_ROLE_ID" \
  secret_id="$VAULT_SECRET_ID"
```

Комментарий: в пайплайнах часто сохраняют только **token** из ответа во временную переменную и дальше вызывают `vault kv get …`.

---

## Kubernetes Auth

Vault доверяет **JWT ServiceAccount** и API Kubernetes. Нужны: доступ Vault к API кластера (или корректная конфигурация для вашей топологии), **CA**, issuer/audience в зависимости от версии и модели токенов.

Best practices:

- узкая роль: `bound_service_account_names`, `bound_service_account_namespaces`, при необходимости **audience**;
- отдельный ServiceAccount на приложение, не `default`;
- мониторинг ошибок аутентификации и ротация kube config на стороне Vault.

### Включение и конфигурация (упрощённый пример)

```bash
vault auth enable kubernetes

# Параметры зависят от окружения: хост API, JWT reviewer, CA
# В production используйте официальный гайд для вашей версии Vault/K8s
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/path/to/ca.pem \
  token_reviewer_jwt=@/path/to/vault-reviewer-token.jwt
```

### Роль для pod

```bash
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp \
  bound_service_account_namespaces=production \
  policies=myapp-kv \
  ttl=1h \
  max_ttl=24h
```

### Логин из pod (после монтирования SA)

```bash
# JWT обычно лежит в /var/run/secrets/kubernetes.io/serviceaccount/token
JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

vault write -field=token auth/kubernetes/login role=myapp jwt="$JWT"
```

Комментарий: в **EKS/GKE/AKS** уточните issuer и настройки **projected SA tokens** (аудитория), чтобы избежать сюрпризов при обновлении платформы.

---

## LDAP / OIDC (SSO)

### LDAP

Подходит, когда пользователи уже в каталоге. Типичная связка: **группы LDAP** → внешние или встроенные group aliases → политики.

```bash
vault auth enable ldap

vault write auth/ldap/config \
  url="ldaps://ldap.example:636" \
  userdn="OU=Users,DC=example,DC=com" \
  groupdn="OU=Groups,DC=example,DC=com" \
  binddn="CN=vault,OU=Service,DC=example,DC=com" \
  bindpass="REDACTED" \
  userattr="sAMAccountName" \
  groupattr="cn" \
  insecure_tls=false
```

Best practice: **LDAPS**, отдельный bind-аккаунт с минимальными правами, пароль bind — из Vault или секрет-менеджера окружения, не в репозитории.

### OIDC (Okta / Keycloak / др.)

Поток: браузерный login → IdP → callback → Vault выдаёт token. Настройки зависят от провайдера (**discovery URL**, client, redirect URIs, scopes).

```bash
vault auth enable oidc

vault write auth/oidc/config \
  oidc_discovery_url="https://keycloak.example/realms/corp/.well-known/openid-configuration" \
  oidc_client_id="vault" \
  oidc_client_secret="REDACTED" \
  default_role="default" \
  oidc_response_mode="query"
```

```bash
vault write auth/oidc/role/default \
  user_claim="sub" \
  allowed_redirect_uris="https://vault.example:8200/ui/vault/auth/oidc/oidc/callback" \
  bound_audiences="vault" \
  oidc_scopes="openid,profile,email" \
  policies="default-read"
```

Best practices:

- **PKCE** и настройки безопасности по гайду HashiCorp + вашего IdP;
- маппинг **групп** из claim (`groups claim`) на политики — отдельная настройка (механизмы identity/group alias);
- для людей — MFA на стороне IdP;
- `oidc_client_secret` в production лучше подставлять из окружения/файла с правами `0400`, не светить в логах.

---

## Сводный production checklist

| # | Практика |
|---|----------|
| 1 | Долгоживущие root/широкие token’ы **не** использовать для приложений и людей |
| 2 | AppRole: разделение ролей по средам, короткий `secret_id`/token TTL |
| 3 | Kubernetes: жёсткие `bound_*`, отдельный SA, актуальная конфигурация JWT |
| 4 | OIDC/LDAP: группы → минимальные политики, аудит логинов |
| 5 | Регулярно пересматривать **auth methods** и отключать неиспользуемые |

---

## Дополнительные материалы

- [Authentication | Vault](https://developer.hashicorp.com/vault/docs/concepts/auth)
- [AppRole auth method](https://developer.hashicorp.com/vault/docs/auth/approle)
- [Kubernetes auth method](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
- [OIDC auth method](https://developer.hashicorp.com/vault/docs/auth/jwt/oidc-providers)

