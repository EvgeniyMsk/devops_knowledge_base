# 1. База: архитектура и базовая практика

---
В этой теме — минимальный набор, чтобы уверенно пользоваться RabbitMQ: ключевые сущности (exchange/queue/binding), типы exchange, базовая отправка/получение сообщений, запуск в Docker и в Kubernetes, а также практики надёжности и настройки для production.

---

## Определения и сущности

| Термин | Определение |
|--------|-------------|
| **Message broker** | Система-посредник для доставки сообщений между сервисами: принимает сообщения от producer’ов и отдаёт consumer’ам, развязывая их по времени и нагрузке. |
| **Producer** | Компонент, который публикует сообщения в broker. |
| **Consumer** | Компонент, который получает и обрабатывает сообщения из очереди. |
| **Exchange** | «Маршрутизатор» сообщений: принимает publish и решает, в какие очереди отправить сообщение по правилам биндингов. |
| **Queue** | Буфер сообщений, из которого потребляют consumer’ы; хранит сообщения до обработки (или до истечения TTL). |
| **Binding** | Правило связи exchange → queue (включая routing key/условие). |
| **Routing key** | Строка маршрутизации при публикации; используется direct/topic exchange для сопоставления биндингам. |
| **Virtual host (vhost)** | Логическая изоляция внутри кластера RabbitMQ: свои exchange/queue/users/permissions. |
| **Connection / Channel** | TCP‑соединение и логический канал внутри него. Обычно держат 1 connection и много channels. |
| **Ack / Nack** | Подтверждение обработки сообщения consumer’ом (ack) или отказ с возможностью requeue/отбрасывания (nack). |
| **Prefetch** | Ограничение количества «неподтверждённых» сообщений на consumer’а; ключ к fair‑dispatch и управлению нагрузкой. |
| **Durable** | Exchange/queue сохраняются при перезапуске broker’а (метаданные). |
| **Persistent message** | Сообщение помечено для записи на диск (вместе с durable queue повышает выживаемость при рестарте). |
| **Publisher confirms** | Механизм подтверждения от broker’а, что публикация принята и сохранена согласно гарантиям. |
| **DLX (Dead Letter Exchange)** | Exchange, куда «переезжают» сообщения при reject/TTL/max-length и т.п.; основа DLQ-паттерна. |
| **Quorum queue** | Очередь на основе Raft‑репликации (HA внутри кластера), предпочтительна для production вместо classic для критичных потоков. |

---

## Архитектура RabbitMQ: путь сообщения

Почти всегда сообщение идёт так:

`producer → exchange → (binding rules) → queue → consumer`

Важно: producer обычно публикует **в exchange**, а не напрямую в queue. Queue «подписывается» на exchange через binding.

---

## Типы exchange (когда какой использовать)

| Тип exchange | Как маршрутизирует | Когда использовать |
|-------------|---------------------|-------------------|
| **direct** | По точному совпадению routing key | Команды/задачи, простая маршрутизация «ключ → очередь». |
| **fanout** | Во все очереди, привязанные к exchange | Broadcast событий (без фильтрации по ключу). |
| **topic** | По шаблону routing key (`*` и `#`) | События с иерархией ключей (`orders.created.eu`), гибкая подписка. |
| **headers** | По заголовкам сообщения | Редко; когда routing key неудобен, а нужны условия по headers. |

Пример topic‑шаблонов:

- `orders.*` — ровно 2 сегмента (`orders.created`, `orders.cancelled`)
- `orders.#` — любое число сегментов (`orders.created.eu.msk`)

---

## Практика: поднять RabbitMQ

### Docker (быстрый старт)

```bash
# Management UI будет на http://127.0.0.1:15672 (логин/пароль: guest/guest)
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management
```

Проверка:

```bash
curl -s http://127.0.0.1:15672 >/dev/null && echo "UI up"
```

### Kubernetes (через Helm)

Один из типичных вариантов — chart `bitnami/rabbitmq` (на практике часто используется в production).

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install rabbitmq bitnami/rabbitmq \
  --namespace rabbitmq --create-namespace \
  --set auth.username=app \
  --set auth.password='change-me' \
  --set auth.erlangCookie='change-me-too'
```

Комментарий:

- `erlangCookie` должен быть одинаковым для всех нод кластера RabbitMQ.
- Пароли/куки в production хранить в Secret и передавать через values/External Secrets, а не в командной строке.

---

## Небольшие примеры кода (AMQP)

### Python: publish (producer) с durability и confirm

```python
import pika

params = pika.ConnectionParameters("127.0.0.1")
conn = pika.BlockingConnection(params)
ch = conn.channel()

# Durable queue: переживёт рестарт (метаданные очереди сохраняются)
ch.queue_declare(queue="tasks", durable=True)

# Включаем publisher confirms: broker подтвердит, что publish принят
ch.confirm_delivery()

body = b'{"task":"resize","id":123}'

ok = ch.basic_publish(
    exchange="",               # default exchange -> routing_key == queue name
    routing_key="tasks",
    body=body,
    properties=pika.BasicProperties(
        delivery_mode=2,       # persistent message
        content_type="application/json",
    ),
)

assert ok, "publish not confirmed"
conn.close()
```

Примечание: публикация в `exchange=""` использует **default exchange** — это удобно для демо, но в production чаще создают свой exchange и биндинги.

### Python: consume (consumer) с ack и prefetch

```python
import pika

params = pika.ConnectionParameters("127.0.0.1")
conn = pika.BlockingConnection(params)
ch = conn.channel()

ch.queue_declare(queue="tasks", durable=True)

# Prefetch: сколько сообщений можно «взять в работу» без ack
ch.basic_qos(prefetch_count=10)

def on_message(ch, method, properties, body):
    try:
        # обработка
        print("got:", body.decode("utf-8"))
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        # Важно: решение requeue зависит от типа ошибки.
        # Для «временных» ошибок можно requeue=True, для «плохих данных» — лучше в DLQ.
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

ch.basic_consume(queue="tasks", on_message_callback=on_message)
ch.start_consuming()
```

---

## CLI-практика (минимум для эксплуатации)

```bash
# Состояние нод и кластера
rabbitmqctl status

# Список очередей и базовые метрики (messages, consumers)
rabbitmqctl list_queues name durable messages_ready messages_unacknowledged consumers

# Список exchange
rabbitmqctl list_exchanges name type durable
```

---

## Production best practices (коротко и по делу)

### Надёжность доставки: что включать

- **Ack обязателен**: автоподтверждение (`autoAck=true`) почти всегда ошибка — при падении consumer’а сообщения теряются.
- **Prefetch на consumer**: ограничивает параллелизм и защищает от «захвата» очереди одним consumer’ом.
- **Durable queue + persistent messages**: базовый минимум для сообщений, которые нельзя терять при рестарте.
- **Publisher confirms**: иначе producer может «думать», что сообщение отправлено, но фактически оно не сохранено.

### Dead Letter Queue (DLQ) и повторные попытки

Типичный production‑паттерн:

- основная очередь `tasks`
- DLQ `tasks.dlq`
- retry‑очередь с TTL `tasks.retry.30s` → возвращает обратно в `tasks` через DLX

```text
tasks  --(reject/ttl/maxlen)-->  dlx.tasks  --> tasks.dlq
tasks  <--(ttl)--- tasks.retry.30s <--(reject)--- consumer
```

Ключевой принцип: **не бесконечно requeue** при постоянной ошибке — это создаёт «горячую» петлю и убивает брокер/consumer’ов.

### Quorum queues вместо classic (если нужен HA)

- Для критичных очередей в production чаще выбирают **quorum queues**: предсказуемее поведение при отказах и репликация на основе Raft.
- Classic очереди проще, но при сетевых сбоях и split brain рискованнее.

### Изоляция и безопасность

- Использовать **vhost’ы** по окружениям/командам (`/prod`, `/staging`) и **минимальные permissions** на уровне vhost.
- Отключать `guest` или хотя бы запрещать ему логин не с localhost.
- Включать TLS на внешних соединениях, а креды хранить в Secret/Vault.

### Наблюдаемость и алерты

- Мониторить: `messages_ready`, `messages_unacknowledged`, рост backlog, consumer count, rate publish/ack.
- Алерты: «очередь растёт N минут», «нет consumer’ов», «высокая доля unacked», «disk free low», «memory alarm».

---

## Дополнительные материалы

- [RabbitMQ — Concepts](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [RabbitMQ — Reliability guide](https://www.rabbitmq.com/reliability.html)
- [RabbitMQ — Quorum Queues](https://www.rabbitmq.com/quorum-queues.html)
