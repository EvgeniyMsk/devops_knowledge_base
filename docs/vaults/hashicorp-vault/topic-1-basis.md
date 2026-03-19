# 1. База: что такое Vault и как он устроен

---
Цель — закрыть обязательный фундамент: что такое **secrets management**, зачем **HashiCorp Vault**, какие бывают типы секретов и как устроены ключевые части продукта (хранилище, seal, токены, политики). Ниже — термины, архитектура, практика с **KV** и **policy**, а также production best practices.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Secrets management** | Управление жизненным циклом секретов: хранение, выдача, ротация, отзыв, аудит, минимальные привилегии. |
| **Vault** | Централизованная платформа HashiCorp для секретов и конфиденциальных данных с API и политиками доступа. |
| **Static secret** | Секрет, который хранится явно (например, пары ключ/значение в **KV**). |
| **Dynamic secret** | Временный секрет, выдаваемый по запросу (например, учётные данные БД, cloud credentials) с TTL. |
| **Storage backend** | Постоянное хранилище состояния Vault (в production чаще **integrated storage / Raft**). |
| **Seal** | Механизм защиты master key: пока Vault «sealed», данные не расшифровываются. |
| **Unseal** | Разблокирование: несколько ключей (**shamir shares**) или cloud KMS auto-unseal. |
| **Token** | Краткоживущий или долгоживущий credential для Vault API с привязкой к политикам и лимитам. |
| **Policy (HCL)** | Правила **что можно** в Vault (`path`, capabilities: `read`, `list`, `create`, `update`, `delete`, `sudo` и т.д.). |
| **Authentication** | Кто вы: userpass, LDAP, Kubernetes auth, AppRole, OIDC… |
| **Authorization** | Что вам разрешено: результат сверки **identity + policies**. |

---

## Зачем Vault, если уже есть Kubernetes Secrets / .env

Production-мотивация:

- **Единая точка** выдачи секретов и **аудит** (кто что читал).
- **Краткоживущие** dynamic credentials вместо «вечного» пароля в конфиге.
- **Политики** и роли вместо общего kubeconfig с доступом ко всему.
- **Ротация** и отзыв через API, а не ручной «поиск по репозиториям».

Best practice: не считать Vault «ещё одной базой паролей» — это **система контроля доступа** к секретам с явной моделью доверия.

---

## Типы секретов: static (KV) и dynamic (DB, cloud)

| Тип | Примеры | Когда использовать |
|-----|---------|-------------------|
| **Static (KV v2)** | API keys, лицензии, статические конфиги | Простые пары ключ/значение, версионирование в KV. |
| **Dynamic** | Credentials к PostgreSQL, IAM для AWS | Временный доступ, автоматическая ротация, минимизация blast radius. |

Комментарий: в production dynamic секреты часто предпочтительнее для БД и облака — меньше «утёкших на годы» паролей.

---

## Архитектура Vault (упрощённо)

1) **Storage** — персистентное хранилище (Raft, ранее часто Consul и др.).
2) **Core** — логика API, плагины **secrets engines** и **auth methods**.
3) **Seal/Unseal** — без unseal Vault не отдаёт данные.
4) **Tokens + Policies** — каждый запрос к API идёт с контекстом политик.

```
Client → Vault API → Auth → Identity + Policies → Secrets Engine → Storage
```

---

## Токены: root, service, batch

| Вид | Назначение | Production |
|-----|------------|------------|
| **Root token** | Полный доступ при инициализации | **Немедленно отозвать** после bootstrap; не использовать в пайплайнах. |
| **Service token** | Обычная работа приложений/людей | Короткий TTL, явные policies, CIDR/metadata при необходимости. |
| **Batch token** | Массовые read-only сценарии, без lease storage | Когда подходят ограничения batch (см. документацию для вашей версии). |

Best practice: **никогда** не хранить root token в Git; выдавать break-glass процедуру и аудит на критические пути.

---

## Политики (HCL): минимальный пример

Разрешить только чтение одного KV-пути приложения:

```hcl
# policy: app-orders-read.hcl
path "secret/data/orders/*" {
  capabilities = ["read", "list"]
}

# Запрет явно опасных путей (дополнительный hardening — опционально)
path "secret/metadata/orders/*" {
  capabilities = ["list", "read"]
}
```

Комментарий: в **KV v2** данные обычно под `secret/data/...`, метаданные — `secret/metadata/...` (точный mount может отличаться).

---

## Практика 1: локальный Vault в **dev** режиме

Только для обучения: in-memory, **не** для production.

```bash
# Запуск (одна нода, auto-unseal в виде dev)
vault server -dev -dev-root-token-id="dev-only-token"

export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='dev-only-token'

vault status
```

Best practice: `-dev` не использовать в staging/prod; отдельный TLS, storage, audit, unseal.

---

## Практика 2: KV secrets engine и пара секретов

После dev старта (или на настроенном кластере с нужным mount):

```bash
# Включить KV v2 на пути secret (если ещё не включён; в -dev часто уже есть mount secret/)
vault secrets enable -path=secret -version=2 kv

# Записать секрет (KV v2)
vault kv put secret/orders/database url="postgres://orders:REDACTED@db:5432/orders"

# Прочитать
vault kv get secret/orders/database

# Версии (KV v2)
vault kv metadata get -mount=secret orders/database
```

Production: значения не логировать; `REDACTED` в примерах — заглушка.

---

## Практика 3: policy + ограниченный token

```bash
# Загрузить политику (файл app-orders-read.hcl как выше)
vault policy write app-orders-read app-orders-read.hcl

# Токен с политикой и коротким TTL
vault token create \
  -policy=app-orders-read \
  -ttl=24h \
  -renewable=true \
  -metadata=service=orders-api
```

Best practices:

- предпочитайте **AppRole / Kubernetes auth / JWT** вместо долгоживущих ручных token’ов,
- TTL и `num_uses` ограничивать по сценарию.

---

## Production mode: чем отличается от dev

| Аспект | Dev | Production |
|--------|-----|------------|
| Storage | in-memory / нет персистентности | Raft (или поддерживаемый backend), backup/DR |
| Unseal | упрощён | Shamir или KMS auto-unseal |
| TLS | часто выкл. | обязательно |
| Audit | часто нет | **audit device** на стабильное хранилище |
| Root token | может быть удобен при dev | отозвать после инициализации |

Комментарий: «production mode» = нормальный кластер с HA (обычно 3–5 узлов Raft), мониторингом, бэкапами snapshot и runbook на seal/unseal.

---

## Production checklist

| Практика | Зачем |
|----------|--------|
| Отозвать root token после bootstrap | снижение критичного blast radius |
| Минимальные policies (least privilege) | меньше утечек при компрометации токена |
| Короткий TTL на токены, auth по ролям | меньше «вечных ключей» |
| Audit log + мониторинг | расследование и соответствие требованиям |
| Не класть секреты в Git | Vault как единственный источник выдачи |

---

## Дополнительные материалы

- [HashiCorp Vault — Documentation](https://developer.hashicorp.com/vault/docs)

