# 2. Надёжность и delivery-паттерны

---
Цель темы — научиться строить «живучую» доставку: не терять сообщения, не умирать от перегрузки и корректно обрабатывать сбои. Здесь собраны ack/nack, publisher confirms, prefetch, TTL, DLX/DLQ и retry‑паттерны с production‑ориентированными примерами.

---

## Ключевые принципы RabbitMQ (важно запомнить)

- **At-least-once** — типовая гарантия: сообщение может прийти **дважды**, поэтому consumer должен быть **идемпотентным**.
- **Ack управляет “когда считать сообщение обработанным”**: без ack вы теряете сообщения при падении consumer’а.
- **Prefetch управляет нагрузкой**: без prefetch один consumer может «схватить» слишком много сообщений и «задушить» себя.
- **Retry — это отдельная инфраструктура**: просто `requeue=true` при ошибке = риск бесконечной горячей петли.

---

## Ack / Nack: как не потерять сообщения

### Что происходит без ack

- `autoAck=true` (или аналог) означает: broker считает сообщение обработанным **сразу при выдаче** consumer’у.
- Если consumer упал в процессе — сообщение уже “пропало” (для очереди оно доставлено).

### Базовый шаблон обработки (manual ack)

```python
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()
ch.queue_declare(queue="tasks", durable=True)

ch.basic_qos(prefetch_count=20)  # ограничиваем параллелизм «в работе»

def handle(body: bytes) -> None:
    # обработка должна быть идемпотентной (возможны дубликаты)
    pass

def on_message(ch, method, properties, body):
    try:
        handle(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        # Важно: решение requeue зависит от причины.
        # Временная ошибка (timeout к БД) -> requeue=True (или retry-очередь)
        # “Плохие данные” -> requeue=False и отправить в DLQ
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

ch.basic_consume(queue="tasks", on_message_callback=on_message, auto_ack=False)
ch.start_consuming()
```

Production‑практика: **логировать причину nack/reject и routing‑метаданные** (message id, correlation id), иначе отладка DLQ будет мучительной.

---

## Durable / Persistent / Confirms: три разные вещи

| Механизм | За что отвечает | Когда нужен |
|---------|------------------|-------------|
| **durable queue/exchange** | Метаданные очереди/эксченджа переживают рестарт | Почти всегда в production |
| **persistent message (delivery_mode=2)** | Сообщение помечается для записи на диск | Для сообщений, которые нельзя терять |
| **publisher confirms** | Producer получает подтверждение от broker’а, что publish принят/сохранён | Для гарантий «producer не молча потерял publish» |

### Producer с confirms (пример)

```python
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()
ch.queue_declare(queue="tasks", durable=True)

ch.confirm_delivery()  # включаем подтверждения

ok = ch.basic_publish(
    exchange="",
    routing_key="tasks",
    body=b'{"id":123}',
    properties=pika.BasicProperties(delivery_mode=2),
)
assert ok, "publish not confirmed"
conn.close()
```

Best practice: при отказе confirms **делать повторную попытку publish** с backoff и **идемпотентным message id**, чтобы не размножать дубликаты.

---

## Prefetch (QoS): как не «убить consumer перегрузкой»

**prefetch_count** — сколько сообщений RabbitMQ может отдать consumer’у без ack (in-flight).

Практика:

- CPU‑bound обработка → небольшой prefetch (1–10)
- I/O‑bound обработка (много ожиданий) → больше (20–200), но измерять по latency и памяти
- Если обработка «длинная», prefetch=1 часто даёт стабильность и предсказуемость

Команда для диагностики:

```bash
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

Если `messages_unacknowledged` растёт без роста `consumers` — consumer’ы не успевают ack.

---

## DLX / DLQ: куда отправлять «проблемные» сообщения

**Dead Letter Exchange (DLX)** — exchange, куда RabbitMQ пересылает сообщения, которые:

- отклонены (`reject` / `nack` с `requeue=false`)
- истекли по TTL
- вытеснены по max-length

Типовой паттерн: основная очередь + DLQ.

```yaml
# Очередь tasks будет отправлять dead-letter в dlx.tasks
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-definitions-snippet
data:
  # Это фрагмент "definitions" (в реальной жизни вы загружаете definitions.json в RabbitMQ).
  # Здесь — идея: у очереди tasks есть x-dead-letter-exchange.
  definitions.json: |
    {
      "queues": [
        {
          "name": "tasks",
          "durable": true,
          "arguments": { "x-dead-letter-exchange": "dlx.tasks" }
        },
        { "name": "tasks.dlq", "durable": true }
      ],
      "exchanges": [
        { "name": "dlx.tasks", "type": "direct", "durable": true }
      ],
      "bindings": [
        { "source": "dlx.tasks", "destination": "tasks.dlq", "destination_type": "queue", "routing_key": "dead" }
      ]
    }
```

Примечание: в реальном Helm‑деплое Bitnami можно подавать definitions через `extraConfiguration` / mounted file (зависит от чарта/версии). Важно именно понимание механики.

---

## Retry‑паттерны (production)

### Антипаттерн: requeue в бесконечность

Если при любой ошибке делать `requeue=true`, получится:

- «горячая» петля (одно и то же сообщение мгновенно возвращается)
- рост нагрузки на broker/consumer
- блокировка очереди проблемными сообщениями

### Паттерн 1: Retry через отдельные очереди с TTL (delayed retry)

Идея:

- consumer при временной ошибке публикует сообщение в `tasks.retry.30s`
- у retry‑очереди задан TTL и DLX обратно в `tasks`

```text
tasks.retry.30s --(TTL expires)--> dlx.retry --> tasks
```

Плюсы: простой, предсказуемый. Минусы: нужно несколько очередей под разные задержки (30s/5m/1h) или отдельный delayed‑plugin.

### Паттерн 2: Retry “счётчиком” в headers

Consumer хранит `x-retries` в headers:

- если `x-retries < N` → отправить в retry‑очередь, увеличить счётчик
- иначе → в DLQ

Важно: producer/consumer должны сохранять headers и message id при репаблише.

---

## TTL: автоочистка и защита от “вечных” сообщений

TTL бывает:

- **message TTL** — срок жизни сообщения
- **queue TTL / expires** — срок жизни очереди (auto-delete)

Production‑идея: для «временных» сообщений (например, “send email within 10 minutes”) TTL защищает от обработки устаревшего.

Риск: TTL + высокая нагрузка = много перемещений в DLX, что может создать пики I/O. TTL применять осознанно и мониторить.

---

## Идемпотентность consumer’ов: обязательный уровень

Причины дубликатов в RabbitMQ:

- consumer обработал, но не успел ack (упал) → broker отдаст сообщение снова
- producer повторил publish после сетевого сбоя (confirm не пришёл) → дубликат

Типовой подход:

- в сообщение добавлять `message_id` (UUID) и/или business key
- в consumer хранить “processed set” в БД/Redis с TTL (например, 24h)
- “exactly-once” достигается не в брокере, а в бизнес‑логике (идемпотентность)

---

## Production best practices (чеклист)

- **ack только после успешной обработки** (и фиксации результата).
- **prefetch обязательно** — иначе рискуете memory‑пиками и несправедливым распределением.
- **publisher confirms** для важных сообщений + повтор publish с backoff.
- **DLQ обязательно** для “плохих” сообщений (schema validation, обязательные поля).
- **Retry через TTL/DLX или delayed‑механизм**, а не бесконечный requeue.
- **Quorum queues** для критичных очередей в HA‑кластере.
- **Наблюдаемость**: backlog (`messages_ready`), in‑flight (`messages_unacknowledged`), rate publish/ack, alarms (memory/disk).

---

## Дополнительные материалы

- [RabbitMQ — Reliability guide](https://www.rabbitmq.com/reliability.html)
- [RabbitMQ — Dead Letter Exchanges](https://www.rabbitmq.com/dlx.html)
- [RabbitMQ — Consumer Prefetch](https://www.rabbitmq.com/consumer-prefetch.html)
