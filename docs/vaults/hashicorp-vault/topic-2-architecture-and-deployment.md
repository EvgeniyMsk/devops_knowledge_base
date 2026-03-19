# 2. Архитектура и развёртывание

---
Цель — уметь **спроектировать и развернуть** Vault для production: выбор топологии (standalone / HA), **Integrated Storage (Raft)** или внешний backend, **TLS**, **seal/unseal**, бэкапы и мышление про отказоустойчивость. Ниже — сравнения, комментарии и короткие примеры конфигурации.

---

## Варианты развёртывания

| Режим | Когда уместен | Production |
|--------|----------------|------------|
| **Standalone** | PoC, dev-лаборатория, очень редкие сценарии с жёстко зафиксированным RPO/RTO | Обычно **нет**: одна точка отказа, нет автоматического лидерства. |
| **HA (несколько узлов)** | Все сценарии с SLA | Стандарт: **3 или 5 узлов** Raft (кворум при потере меньшинства узлов). |

Best practice: даже в «внутренней» среде целиться в HA, если Vault — критический контур (аутентификация приложений, корни доверия).

---

## Хранилище: Integrated Storage (Raft) и Consul

| Backend | Суть | Комментарий для production |
|---------|------|----------------------------|
| **Integrated Storage (Raft)** | Встроенное HA-хранилище на узлах Vault | **Рекомендуемый путь** для новых установок: меньше движущихся частей, snapshot’ы через `vault operator raft snapshot`. |
| **Consul как storage** | Внешний кластер Consul под state Vault | Legacy/наследие; новые проекты чаще выбирают Raft, если нет жёсткой причины держать Consul. |

Критично: **отдельно планируйте диски** под Raft (низкая задержка, надёжный volume, мониторинг свободного места и I/O).

---

## Seal и unseal

Пока кластер **sealed**, операционные данные не расшифровываются — API для работы с секретами недоступен (в типичной конфигурации).

### Shamir Secret Sharing

- При **инициализации** вы получаете **несколько unseal key shares**; для unseal нужно **threshold** из **total** ключей.
- Best practice: ключи **раздельно хранить** (разные люди / сейф / break-glass процедура), **не** в одном чате и **не** в Git.

### Auto-unseal (облачные KMS)

| Провайдер | Механизм в Vault (типично) | Комментарий |
|-----------|------------------------------|-------------|
| **AWS** | `awskms` | IAM с минимальными правами на `kms:Decrypt`/`Encrypt` для CMK. |
| **GCP** | `gcpckms` | Сервисный аккаунт / workload identity, отдельный key ring. |
| **Azure** | `azurekeyvault` | Предпочтительно **managed identity** / workload identity вместо длинноживущих client secrets в конфиге. |

Best practices:

- auto-unseal **не заменяет** резервное копирование и runbook на полную потерю KMS/ключей;
- после миграции с Shamir на auto-unseal зафиксируйте **recovery keys** / процедуру регенерации согласно документации вашей версии Vault.

---

## TLS в production

| Практика | Зачем |
|----------|--------|
| **TLS на listener** | Конфиденциальность токенов и ответов API в сети. |
| **Доверенные сертификаты** | Корпоративный PKI или Let's Encrypt + автоматическая ротация. |
| **`api_addr` / `cluster_addr` с HTTPS** | Корректная реклама адресов для Raft/API; избегать «смешанного» HTTP при срочном debug. |

Комментарий: завершение TLS на reverse-proxy допустимо, если вы **сохраняете** end-to-end модель доверия (SNI, health checks, таймауты) и не ломаете **cluster** трафик между узлами.

---

## Резервное копирование и аварийное восстановление

| Тема | Production-подход |
|------|-------------------|
| **Snapshot (Raft)** | Регулярные `vault operator raft snapshot save` (или snapshot agent) → хранилище **вне** кластера Vault, с **шифрованием at-rest** бэкапа и контролем доступа. |
| **Restore** | Тестируйте **восстановление на стенде** (drill): полный цикл «snapshot → новый кластер → unseal/auto-unseal». |
| **DR** | Для географического DR смотрите официальные модели (**replication** и требования к версии/лицензии); это отдельное решение от «просто бэкапа файлов». |

Best practice: RTO/RPO зафиксировать **до** инцидента; бэкап без теста восстановления считается ненадёжным.

---

## Пример: узел HA-кластера с Raft и TLS

Фрагмент `vault.hcl` (адаптируйте пути, `node_id`, адреса и имена файлов). **Один узел** — шаблон; на остальных меняются `node_id`, `api_addr`/`cluster_addr` и блок присоединения к кворуму.

```hcl
# Пример production-ориентированной конфигурации (не копировать без адаптации)
ui = true

listener "tcp" {
  address         = "0.0.0.0:8200"
  tls_cert_file   = "/etc/vault/tls/vault.crt"
  tls_key_file    = "/etc/vault/tls/vault.key"
  tls_min_version = "tls12"
}

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-1"

  # Присоединение к уже существующему лидеру (пример статического списка)
  retry_join {
    leader_api_addr = "https://vault-2.internal:8200"
  }
  retry_join {
    leader_api_addr = "https://vault-3.internal:8200"
  }
}

api_addr     = "https://vault-1.internal:8200"
cluster_addr = "https://vault-1.internal:8201"
```

Комментарий: `retry_join` бывает **static**, **auto** (облако/Kubernetes) — выбирайте то, что соответствует вашему IaaS; главное — стабильный DNS/IP для лидера при bootstrap.

---

## Пример: auto-unseal через AWS KMS

```hcl
seal "awskms" {
  region     = "eu-central-1"
  kms_key_id = "alias/prod-vault-unseal" # или полный ARN ключа
  # endpoint / role ARN — при необходимости для VPC endpoints и явной роли
}
```

Best practice: отдельный **CMK**, отдельный IAM role на узел / на pod, минимальные действия KMS, **CloudTrail** на ключ.

---

## Пример: auto-unseal через GCP Cloud KMS

```hcl
seal "gcpckms" {
  project    = "my-gcp-project"
  region     = "europe-west3"
  key_ring   = "vault-prod"
  crypto_key = "vault-unseal"
}
```

Учётные данные — через **сервисный аккаунт** с правами на unwrap key, без ключей JSON в образе контейнера (Workload Identity / metadata).

---

## Пример: auto-unseal через Azure Key Vault

```hcl
seal "azurekeyvault" {
  tenant_id = "00000000-0000-0000-0000-000000000000"
  vault_name = "kv-prod-vault-unseal"
  key_name   = "vault-unseal"
  # В production: аутентификация через managed identity (см. документацию Vault для вашей версии)
}
```

Комментарий: **не** храните `client_secret` в репозитории; для VM используйте **system-assigned identity**, для AKS — **workload identity**.

---

## Оперативные команды (Raft)

```bash
# Статус узла
vault status

# Участники Raft-кластера (на разрешённом по policy операторе)
vault operator raft list-peers

# Снимок состояния (делайте по расписанию и выгружайте off-cluster)
vault operator raft snapshot save /secure/backups/vault-$(date +%Y%m%d-%H%M).snap
```

---

## Краткий production checklist

| # | Проверка |
|---|----------|
| 1 | HA Raft, **не** одиночный узел без обоснования |
| 2 | TLS, корректные `api_addr` / `cluster_addr` |
| 3 | Выбран и задокументирован **seal** (Shamir и/или KMS) |
| 4 | Регулярные **snapshot** + тест restore |
| 5 | Audit device и мониторинг (лидер, sealed, latency storage) |
| 6 | Runbook: потеря кворума, потеря региона, ротация TLS |

---

## Дополнительные материалы

- [Vault storage backends](https://developer.hashicorp.com/vault/docs/internals/storage)
- [Integrated storage (Raft)](https://developer.hashicorp.com/vault/docs/configuration/storage/raft)
- [Seal configuration](https://developer.hashicorp.com/vault/docs/configuration/seal)

