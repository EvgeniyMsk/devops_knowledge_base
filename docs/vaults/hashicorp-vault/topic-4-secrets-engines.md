# 4. Secrets engines

---
Цель — **уверенно пользоваться движками секретов**: статические **KV v2**, **динамические** учётные записи (БД, облако), **PKI** для сертификатов и **Transit** для шифрования и подписи без хранения ключевого материала у приложения. Ниже — суть каждого движка, production-замечания и короткие примеры.

---

## Общая идея

**Secrets engine** монтируется на путь (`path`). Запросы к API по этому пути обрабатывает плагин (KV, database, `aws`, `pki`, `transit`…). Доступ регулируется **политиками** на конкретные `path`-ы.

Best practice: **отдельные mount’ы** по средам или доменам (`secret/`, `kv_prod/`) и узкие политики — проще расследовать утечки и делегировать владение.

---

## KV v2 (Key/Value)

| Возможность | Смысл |
|-------------|--------|
| **Versioning** | Каждое обновление — новая версия; можно откатиться на предыдущую. |
| **Soft delete** | Версия помечается удалённой; при необходимости — **undelete** или окончательное **destroy**. |

Production:

- не класть **в KV** огромные бинарники — это не object storage;
- явно решить политику **destroy** vs **undelete** (комплаенс, GDPR);
- бэкап Raft покрывает и KV; всё равно иметь runbook на случай порчи данных.

### Примеры CLI

```bash
vault kv put secret/apps/payments/config api_key="REDACTED" region="eu-central-1"

vault kv get secret/apps/payments/config

# Версии и метаданные
vault kv metadata get -mount=secret apps/payments/config

# «Мягкое» удаление последней версии (путь может отличаться в вашей версии CLI)
vault kv delete -mount=secret apps/payments/config

# Восстановление (если движок и политика позволяют)
vault kv undelete -mount=secret -versions=3 apps/payments/config

# Окончательное удаление версии — необратимо в рамках движка
vault kv destroy -mount=secret -versions=2 apps/payments/config
```

Комментарий: в политиках KV v2 часто нужны и `secret/data/...`, и `secret/metadata/...` (в зависимости от операций).

---

## Dynamic secrets: Database (PostgreSQL / MySQL)

Vault **создаёт временного пользователя** в БД по шаблону SQL и **отзывает** по истечении lease.

Best practices:

- отдельный **DB user для Vault** только с правами `CREATE ROLE` / выдачи прав (минимально необходимые);
- **короткий default_ttl** и узкие `creation_statements` / `revocation_statements`;
- мониторинг **отзыва** (failed revocation → ручной cleanup в БД);
- в production строку подключения и пароль административной роли — не хардкодить в репозиторий.

### PostgreSQL: конфигурация и роль

```bash
vault secrets enable -path=database database

vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db.internal:5432/appdb?sslmode=require" \
  allowed_roles="readonly","app-writer" \
  username="vault_admin" \
  password="REDACTED"
```

```bash
# Роль: выдача read-only пользователя на 1 час
vault write database/roles/readonly \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

```bash
# Получить динамические креды (имя пользователя и пароль в ответе)
vault read database/creds/readonly
```

Для **MySQL** меняется `plugin_name`, `connection_url` и SQL в `creation_statements` — держитесь официального примера HashiCorp под вашу версию.

---

## Dynamic secrets: AWS IAM

Vault выдаёт **временные ключи** или **роли через STS** (зависит от настройки роли в Vault).

Best practices:

- **не** использовать root аккаунт AWS в `aws/config/root`; отдельный IAM user/role с минимальным `sts:AssumeRole` / политикой выдачи;
- короткий TTL, отдельные **Vault role** на каждый класс доступа;
- аудит CloudTrail + Vault audit по путям `aws/creds/...`.

### Упрощённый пример

```bash
vault secrets enable -path=aws aws

vault write aws/config/root \
  access_key="REDACTED" \
  secret_key="REDACTED" \
  region="eu-central-1"
```

```bash
# Роль: выдача временных кредов с политикой IAM (см. документацию по credential_type)
vault write aws/roles/s3-reader \
  credential_type=iam_user \
  policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::corp-reports/*" }
  ]
}
EOF
```

```bash
vault read aws/creds/s3-reader
```

Комментарий: в production чаще переводят Root config на **STS/assumed role** вместо долгоживущих ключей — см. актуальный гайд HashiCorp.

---

## PKI

Сценарии: **внутренний CA**, TLS для сервисов, короткоживущие leaf-сертификаты, автоматическая ротация.

Архитектура:

1. **Root CA** (узкий круг операций, длинный срок жизни или офлайн).
2. **Intermediate CA** (подписан root; на нём повседневная выдача).
3. **Роль (role)** + **issue** leaf для сервисов.

Best practices:

- **intermediate** на отдельном mount/path;
- ограничить `allowed_domains`, TTL issuance, CRL/OCSP по требованиям;
- защитить root: отдельный кластер/процедуры, минимум онлайн-операций.

### Короткая схема команд (учебный контур)

```bash
# Root PKI
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

vault write pki/root/generate/internal \
  common_name="Corp Root CA" \
  ttl=87600h \
  issuer_name="corp-root"

# Intermediate
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int

vault write -format=json pki_int/intermediate/generate/internal \
  common_name="Corp Intermediate CA" \
  | jq -r '.data.csr' > /tmp/pki_int.csr

vault write -format=json pki/root/sign-intermediate csr=@/tmp/pki_int.csr \
  format=pem_bundle ttl=43800h \
  | jq -r '.data.certificate' > /tmp/signed_intermediate.crt

vault write pki_int/intermediate/set-signed certificate=@/tmp/signed_intermediate.crt
```

```bash
# Роль и выдача сертификата для сервиса
vault write pki_int/roles/internal-service \
  allowed_domains="svc.cluster.local,internal.example" \
  allow_subdomains=true \
  max_ttl="720h"

vault write pki_int/issue/internal-service \
  common_name="api.payments.svc.cluster.local" \
  ttl="24h"
```

Комментарий: в production добавьте **ACME**, автообновление через агенты/VSO и политики на `issue`/`sign` по сервисным identity.

---

## Transit engine

**Шифрование и подпись** ключами Vault; приложение хранит только **ciphertext** или проверяет подпись, не владея ключом.

| Операция | Применение |
|----------|------------|
| **Encrypt / decrypt** | PII в БД, поля в событиях |
| **Sign / verify** | JWT-подобные артефакты, аудит целостности |

Best practices:

- ключи с политикой **только_encrypt** для сервисов, которым не нужен decrypt на том же токене;
- алгоритмы и ротация ключей (`latest` vs версия);
- не передавать ключевой материал клиенту — только API Vault.

### Примеры

```bash
vault secrets enable transit

vault write -f transit/keys/app-data type=aes256-gcm96

# plaintext должен быть base64
vault write transit/encrypt/app-data \
  plaintext=$(echo -n 'user:424242' | base64)

# Расшифровка (нужна политика с update на decrypt)
vault write transit/decrypt/app-data ciphertext="vault:v1:REDACTED_CIPHERTEXT"
```

```bash
vault write -f transit/keys/api-jwt type=rsa-4096

vault write transit/sign/api-jwt input=$(echo -n 'payload' | base64) hash_algorithm=sha2-256

vault write transit/verify/api-jwt input=$(echo -n 'payload' | base64) hash_algorithm=sha2-256 signature="vault:v1:REDACTED_SIG"
```

---

## Сводный production checklist по движкам

| Движок | На что смотреть |
|--------|------------------|
| **KV** | версии, destroy, размер значений, политики data/metadata |
| **Database** | TTL, отзыв, права vault_admin в БД, SSL к БД |
| **AWS** | не root, минимальные IAM, STS где возможно |
| **PKI** | root/intermediate, домены, TTL, CRL, доступ к `sign` |
| **Transit** | разделение encrypt/decrypt, версии ключей, алгоритмы |

---

## Дополнительные материалы

- [Secrets engines](https://developer.hashicorp.com/vault/docs/secrets)
- [KV v2](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2)
- [Database secrets engine](https://developer.hashicorp.com/vault/docs/secrets/databases)
- [AWS secrets engine](https://developer.hashicorp.com/vault/docs/secrets/aws)
- [PKI secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki)
- [Transit secrets engine](https://developer.hashicorp.com/vault/docs/secrets/transit)

