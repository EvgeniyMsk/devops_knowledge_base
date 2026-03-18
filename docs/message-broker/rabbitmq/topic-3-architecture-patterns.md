# 3. Паттерны применения и архитектура

---
Эта тема — про то, **как проектировать messaging‑архитектуру** на RabbitMQ: очередь задач (work queue), pub/sub, RPC поверх очередей, event-driven подход и обязательная идемпотентность. Везде — короткие примеры кода и production‑практики.

---

## Быстрый выбор паттерна (шпаргалка)

| Нужно | Паттерн | Как в RabbitMQ |
|------|---------|----------------|
| Асинхронно выполнить работу | **Work queue** | Одна очередь `tasks`, несколько consumer’ов, prefetch, manual ack |
| Разослать событие многим | **Pub/Sub** | `fanout` exchange → несколько очередей подписчиков |
| Селективная подписка по типу события | **Topic-based pub/sub** | `topic` exchange + routing keys (`orders.created`) |
| “Запрос‑ответ” через брокер | **RPC over RabbitMQ** | reply queue + `correlation_id`, timeout, idempotency |
| Интеграция доменных событий | **Event-driven** | контракт событий, versioning, DLQ, replay стратегия |

Важно: если нужна **строгая гарантия порядка** глобально или очень высокий throughput с партиционированием, RabbitMQ может быть не лучшим выбором (Kafka/NATS JetStream — по задаче). Для большинства интеграций и асинхронных задач RabbitMQ отлично подходит.

---

## Work queues: очередь задач (job queue)

### Когда использовать

- «Сделай работу и не блокируй API»: ресайз изображений, отправка email, генерация отчётов, фоновая обработка.

### Базовая схема

`producer → exchange (direct/default) → tasks queue → N consumers`

### Production‑настройки (минимум)

- **manual ack** (ack после успешной обработки)
- **prefetch** (fair dispatch)
- **durable queue + persistent messages**
- **DLQ** для “плохих” задач и ограничение retry

### Мини‑пример (Python consumer)

```python
import pika
import json

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()
ch.queue_declare(queue="tasks", durable=True)
ch.basic_qos(prefetch_count=10)

def on_message(ch, method, props, body):
    job = json.loads(body.decode("utf-8"))
    try:
        # обработка job
        ch.basic_ack(method.delivery_tag)
    except Exception:
        # “плохие” данные -> DLQ (requeue=False)
        ch.basic_nack(method.delivery_tag, requeue=False)

ch.basic_consume("tasks", on_message_callback=on_message, auto_ack=False)
ch.start_consuming()
```

Best practice: если задачи долгие/дорогие — добавляйте **heartbeat/timeout**, а в самих задачах храните **статус** в БД, чтобы ретраи не выполняли работу дважды.

---

## Pub/Sub: широковещательные события

### Когда использовать

- «Событие произошло, многим нужно узнать»: `user.created`, `order.paid`, `invoice.issued`.

### Схема в RabbitMQ

Fanout exchange:

`producer → exchange(fanout) → queue_A → consumer_A`  
`producer → exchange(fanout) → queue_B → consumer_B`

Каждый подписчик должен иметь **свою очередь**, иначе один consumer «съест» события другого (это будет work queue, а не pub/sub).

### Пример (концептуально)

```text
exchange: events.fanout (type=fanout, durable)
queues:
  billing.events
  analytics.events
bindings:
  events.fanout -> billing.events
  events.fanout -> analytics.events
```

Production best practices:

- События должны иметь **контракт** (schema), версию (`event_version`) и метаданные (`event_id`, `occurred_at`).
- У каждого consumer’а — **DLQ** и политика retry.
- Не публиковать PII/секреты в события «как есть».

---

## Topic routing: селективная подписка

### Когда использовать

- Разные consumer’ы хотят получать **часть** событий по шаблону: например, `orders.*` или `payments.#`.

### Схема

`producer → exchange(topic) → queues (bindings по patterns)`

Пример:

- Billing подписывается на `orders.paid`
- Analytics на `orders.#`

Production best practices:

- Нормализуйте routing keys: `domain.event.action` (`orders.created`, `payments.failed`).
- Не делайте routing key «свалкой» (слишком много сегментов, сложно сопровождать).

---

## RPC over RabbitMQ (request/reply)

### Когда использовать (редко)

RPC через брокер имеет смысл, когда:

- нет прямой сети между сервисами, но есть доступ к брокеру
- нужна очередность/буферизация запросов

Во многих случаях проще HTTP/gRPC. Если используете RPC — обязательно добавляйте timeouts и защиту от утечек reply‑очередей.

### Базовая идея

Producer отправляет request с:

- `reply_to`: очередь для ответов
- `correlation_id`: идентификатор запроса

Consumer отвечает в `reply_to`, копируя `correlation_id`.

### Мини‑пример (Python client)

```python
import pika
import uuid

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()

result = ch.queue_declare(queue="", exclusive=True)  # временная reply queue
callback_queue = result.method.queue
corr_id = str(uuid.uuid4())

response = None

def on_response(ch, method, props, body):
    global response
    if props.correlation_id == corr_id:
        response = body

ch.basic_consume(queue=callback_queue, on_message_callback=on_response, auto_ack=True)

ch.basic_publish(
    exchange="",
    routing_key="rpc.queue",
    properties=pika.BasicProperties(
        reply_to=callback_queue,
        correlation_id=corr_id,
        delivery_mode=2,
    ),
    body=b"ping",
)

# В production тут должен быть timeout + event loop.
while response is None:
    conn.process_data_events(time_limit=1)

print("got:", response)
conn.close()
```

Production best practices:

- **Timeout** обязателен, иначе клиент может зависнуть навсегда.
- Reply queue должна быть **exclusive + auto-delete** (или server‑named), чтобы не накапливать мусор.
- Consumer должен быть **идемпотентным** — клиент может повторить RPC при таймауте.

---

## Event-driven архитектура: как делать «по-взрослому»

### Событие vs команда

- **Command** (команда) — “сделай”: одна цель, один владелец, чаще work queue.
- **Event** (событие) — “произошло”: множество подписчиков, pub/sub.

### Versioning и совместимость

Best practice:

- добавляйте поля только **backward compatible**
- не удаляйте поля резко; делайте миграции в несколько шагов
- используйте `event_type` + `event_version` и schema registry (хотя бы в виде репозитория со схемами)

### Дедупликация и идемпотентность

Идемпотентность — обязательна для at-least-once:

- в событиях: `event_id` (UUID)
- в команде/задаче: `job_id` или business key
- consumer хранит “processed ids” (БД/Redis) и пропускает дубликаты

Простой пример дедупа (псевдо):

```text
if redis.setnx(event_id, 1) == 0:  # уже было
  ack
else:
  redis.expire(event_id, 86400)
  handle(event)
  ack
```

---

## Production best practices (проектирование)

- **Выделяйте vhost по окружениям** (`/prod`, `/staging`) и разделяйте права.
- **Одна очередь на одного logical consumer group** (для pub/sub — очередь на каждого подписчика).
- **Ограничивайте ретраи** (max attempts + DLQ), не допускайте бесконечных requeue.
- **Измеряйте backlog и latency**: рост `messages_ready` = сигнал, что система не успевает.
- **Тестируйте схемы отказов**: падение consumer’ов, рестарт брокера, сеть/timeout к БД.

---

## Дополнительные материалы

- [RabbitMQ — Tutorials](https://www.rabbitmq.com/getstarted.html)
- [RabbitMQ — Reliability guide](https://www.rabbitmq.com/reliability.html)
