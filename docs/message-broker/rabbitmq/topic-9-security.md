# 9. Security

---
Тема про безопасность RabbitMQ в production: изоляция через **vhost**, управление **users/permissions**, TLS для AMQP и management UI, а также варианты аутентификации (встроенная, LDAP/OIDC через прокси/плагины — по окружению). Цель — минимизировать blast radius и исключить “guest/guest в интернет”.

---

## Базовые сущности безопасности

| Сущность | Что даёт |
|---------|----------|
| **User** | Учётная запись для подключения к RabbitMQ (AMQP/HTTP API). |
| **Virtual host (vhost)** | Логическая изоляция: свои exchange/queues/bindings, свои права. |
| **Permissions** | Разрешения на уровне vhost: configure/write/read (часто в виде regex). |
| **TLS** | Шифрование трафика и (опционально) взаимная аутентификация (mTLS). |
| **Auth backend** | Источник аутентификации/авторизации (встроенный, LDAP и т.д.). |

---

## Users / vhosts / permissions: изоляция и least privilege

### Best practice: vhost по окружению и/или домену

Например:

- `/prod`
- `/staging`
- `/team-a`

Это уменьшает риск, что сервис из staging случайно читает очереди production.

### Permissions: configure/write/read

Права делятся на:

- **configure** — создание/изменение exchange/queues/bindings
- **write** — publish (писать в exchange)
- **read** — consume (читать из queues)

Production подход:

- приложениям обычно **не давать configure** (топологию создаёт платформа/CI)
- producer’ам — write, consumer’ам — read
- права ограничивать по именам ресурсов (regex), а не “всё подряд”

### CLI‑пример (концептуально)

```bash
# Создать vhost
rabbitmqctl add_vhost /prod

# Создать пользователя
rabbitmqctl add_user app_producer 'change-me'

# Выдать права: без configure, write только на events.*, read запрещён
rabbitmqctl set_permissions -p /prod app_producer "" "^events\\." ""
```

Комментарий:

- Это пример логики. Regex‑политики надо согласовывать со схемой имён exchange/queue.
- Секреты хранить в Secret/Vault, а не в истории терминала.

---

## TLS: где включать и что защищать

### Что шифровать

- AMQP порт (обычно 5672 → 5671 для TLS, зависит от конфигурации)
- Management UI / HTTP API (15672) — обязательно за TLS и auth

### Практика в Kubernetes

- TLS чаще терминируют на Ingress (для UI/API), а AMQP — через Service типа LoadBalancer/NodePort с TLS на самом RabbitMQ (или через mesh/sidecar — по архитектуре).
- Внутрикластерный трафик тоже стоит шифровать, если есть требования compliance/Zero Trust.

Production best practices:

- запретить “голый” доступ к management UI снаружи
- ограничить доступ по сети (NetworkPolicy / firewall)
- регулярно ротировать сертификаты (cert-manager, Vault PKI)

---

## Auth backends: что бывает

| Вариант | Когда использовать |
|--------|-------------------|
| **Встроенная аутентификация RabbitMQ** | Базовый вариант, простой и часто достаточный; важна дисциплина по пользователям и ротации секретов. |
| **LDAP** | Когда есть централизованный каталог и требуется единый контроль доступа. |
| **OIDC/SSO для UI** | Часто реализуют через reverse proxy перед management UI (nginx/oauth2-proxy), чтобы не раздавать логины RabbitMQ людям. |

Важно: для приложений чаще всего всё равно остаются service users (non-human) с ограниченными permissions.

---

## Обязательные production-ограничения

- **Отключить/ограничить guest**:
  - запретить ему доступ не с localhost
  - или удалить/заменить на управляемых пользователей
- **Не давать приложениям admin‑права**.
- **Сетевые ограничения**:
  - доступ к AMQP только из нужных namespace/подсетей
  - UI/API — только через VPN/SSO/allowlist
- **Audit и учёт изменений**:
  - кто создаёт очереди/exchange (желательно через GitOps/CI, а не “руками”)

---

## Небольшой пример: разделение producer и consumer пользователей

```text
vhost: /prod
user: orders_producer  -> write to exchange orders.*
user: orders_consumer  -> read from queue orders.worker
```

Это снижает blast radius: producer не может вычитывать чужие сообщения, а consumer не может публиковать куда угодно.

---

## Дополнительные материалы

- [RabbitMQ — Access Control](https://www.rabbitmq.com/access-control.html)
- [RabbitMQ — TLS](https://www.rabbitmq.com/ssl.html)
- [RabbitMQ — Authentication/Authorization](https://www.rabbitmq.com/access-control.html)
