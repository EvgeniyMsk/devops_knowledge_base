# 5. Безопасность

---

Kafka в production нужно защищать по трём слоям:

- **аутентификация** (кто ты?) — `SASL` (PLAIN, SCRAM, OAUTH)
- **шифрование** (как передаётся?) — `TLS`
- **авторизация** (что ты можешь?) — `ACL`

Ниже — практический production-подход: минимально необходимые настройки + как правильно обращаться с сертификатами и секретами.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Principal** | Идентичность (пользователь/сервис) в Kafka security модели. |
| **SASL** | Механизм аутентификации (pluggable), например SCRAM или OAuth. |
| **SCRAM** | SASL-механизм на базе challenge-response с паролем (Salted/hashed). |
| **PLAIN** | Простой SASL-механизм (используйте только вместе с TLS, иначе это небезопасно). |
| **OAUTH** | Аутентификация на основе токенов (OIDC/JWT) вместо статического пароля. |
| **TLS** | Шифрование трафика + валидация сертификатов (серверные и клиентские цепочки). |
| **ACL** | Правила доступа: кто (principal) на что (ресурс) и чем (операции) имеет право. |

---

## Аутентификация: SASL (PLAIN, SCRAM, OAUTH)

### SASL PLAIN

Когда использовать:

- только в связке с `TLS`,
- когда SCRAM/UAuth пока не готовы.

Production best practices:

- не используйте PLAIN без TLS,
- ограничьте доступ по сеть/Ingress + TLS,
- храните пароли/секреты в Vault/secret store, а не в конфигурации.

### SASL SCRAM

Плюсы:

- стабильный и широко поддерживаемый production-механизм,
- проще расследовать доступ, чем с “токенами в вакууме”.

Min-setup:

- настроить SCRAM users (в Kafka — via соответствующей утилиты),
- включить `listeners`/`security.protocol` и SASL механизм.

### SASL OAUTH (через OIDC/JWT)

Плюсы:

- единая модель identity через IAM,
- ротация токенов без “ручного перезапуска” паролей.

Production best practices:

- используйте короткоживущие токены и корректные claims,
- проверяйте clock skew (время) на всех компонентах,
- заранее определяйте policy маппинга claim’ов -> principal.

---

## Шифрование: TLS (включая TLS между broker’ами)

### Что обязательно определить в настройке

1) Как брокер слушает соединения (какой `listener`):

- internal/inter-broker listener (для broker↔broker),
- client listener (для producer/consumer).

2) Где лежат сертификаты и ключи:

- keystore,
- truststore,
- CA цепочка,
- и требование к клиентской аутентификации (mTLS).

### Пример: фрагмент `server.properties` (idea)

```properties
# Для inter-broker связи важно задавать отдельный listener
inter.broker.listener.name=INTERNAL

listener.security.protocol.map=INTERNAL:SASL_SSL,CLIENT:SASL_SSL

listeners=INTERNAL://0.0.0.0:9093,CLIENT://0.0.0.0:9092

# TLS settings (пример)
ssl.keystore.location=/etc/kafka/tls/keystore.p12
ssl.keystore.password=${KEYSTORE_PASSWORD}
ssl.key.password=${KEY_PASSWORD}

ssl.truststore.location=/etc/kafka/tls/truststore.p12
ssl.truststore.password=${TRUSTSTORE_PASSWORD}

# production: часто включают mTLS для внутреннего транспорта
ssl.client.auth=required
```

Комментарий:

- точные пути/форматы сертификатов зависят от того, как вы получаете и монтируете certs,
- пароли ключей должны приходить из секретов, а не из plain-text.

---

## Авторизация: ACL (кто что может)

ACL — это правила вида:

- principal (кто),
- ресурс (что),
- операции (что именно: чтение/запись/обслуживание),
- при необходимости — условие по consumer group’ам/кластерам.

### Production best practices для ACL

- принцип наименьших привилегий (least privilege),
- разделяйте роли сервисов (producer’ы отдельно от consumer’ов),
- ограничивайте операции строго по нужным сценариям,
- не “давайте everything” на время отладки: вместо этого делайте temporary policy и удаляйте.

### Мини-пример: выдача прав через kafka-acls

Утилита и формат команды зависят от вашей версии Kafka. Ниже — шаблон логики:

```bash
# Важно: используйте principal’ы, которые реально соответствуют вашему SASL механизму
kafka-acls.sh \
  --bootstrap-server kafka-0.kafka:9092 \
  --add \
  --principal "User=orders-api" \
  --operation Describe \
  --resource-type <RESOURCE_KIND> \
  --resource-name <RESOURCE_NAME_PATTERN> \
  --group <OPTIONAL_GROUP_FOR_CONSUMERS>
```

Комментарий:

- вместо `<RESOURCE_KIND>`/`<RESOURCE_NAME_PATTERN>` подставьте ваши реальные значения,
- для consumer’ов обычно нужны дополнительные права на group/offset-подобные операции.

---

## DevOps-фокус: secrets management (Vault / IAM / ротация)

### Принципы

- сертификаты и пароли не должны жить в Git,
- ротация не должна требовать ручного вмешательства на каждый broker,
- одинаковые секреты должны быть доступны и internal (mTLS), и client listeners.

### Пример: initContainer для загрузки TLS-артефактов

Идея: на старте pod скачивает сертификаты/ключи из Vault и пишет их в emptyDir.

```yaml
volumes:
  - name: tls
    emptyDir: {}

initContainers:
  - name: fetch-certs
    image: hashicorp/vault:1.18
    command: ["sh", "-c"]
    args:
      - >
        vault kv get -field=keystore.p12 secret/kafka/tls
        > /tls/keystore.p12 &&
        vault kv get -field=truststore.p12 secret/kafka/tls
        > /tls/truststore.p12
    volumeMounts:
      - name: tls
        mountPath: /tls

containers:
  - name: kafka
    volumeMounts:
      - name: tls
        mountPath: /etc/kafka/tls
```

Комментарий:

- в реальном проекте вам понадобится auth к Vault (Kubernetes auth/JWT/OIDC),
- для production лучше управлять доступом через отдельные service account’ы и минимальные политики Vault.

---

## Production checklist

| Что сделать | Зачем |
|--------------|-------|
| Включить TLS для всех listener’ов | защита данных в transit |
| Защитить internal broker↔broker mTLS (где возможно) | уменьшение поверхности атаки |
| Использовать SCRAM или OAuth вместо PLAIN | сильнее security-профиль |
| Ввести least privilege ACL по сервисам | снижение blast radius |
| Секреты и certs держать в Vault/IAM и ротировать | исключить “вечные” ключи |

