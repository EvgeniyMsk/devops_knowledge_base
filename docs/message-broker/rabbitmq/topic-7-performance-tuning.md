# 7. Performance и тюнинг

---
Задача темы — понимать, **где узкие места** RabbitMQ и как тюнить систему под нагрузку: баланс throughput/latency, лимиты соединений и каналов, prefetch, “lazy” поведение очередей, и как делать нагрузочные тесты так, чтобы результаты были полезными. Всё — с практическими командами и production best practices.

---

## Throughput vs Latency: что оптимизируем

- **Throughput** — сколько сообщений/сек система переваривает.
- **Latency** — задержка доставки/обработки сообщения.

Частый trade‑off:

- увеличиваем prefetch/батчи → растёт throughput, но может вырасти latency и memory
- жёстко ограничиваем in‑flight → падает throughput, но latency становится предсказуемее

Production подход: сначала определить SLO (например, “99% сообщений обработать < 2s”), а потом тюнить под него.

---

## Главные рычаги производительности

### 1) Connections / Channels: не убивать брокер соединениями

Правило: **один процесс приложения держит 1 connection** и несколько channels.

Антипаттерн:

- connection per message
- connection per request

Симптомы:

- рост connections/channels
- скачки CPU и latency

CLI‑проверка:

```bash
rabbitmqctl list_connections name user peer_host peer_port state channels
rabbitmqctl list_channels connection pid number consumer_count messages_unacknowledged
```

Мини‑пример (Python): переиспользование connection/channel

```python
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()

for i in range(10000):
    ch.basic_publish(exchange="", routing_key="tasks", body=f"job-{i}".encode())

conn.close()
```

### 2) Prefetch tuning: контроль in‑flight

Prefetch — основной регулятор нагрузки на consumer.

Рекомендации:

- CPU‑bound → 1..10
- I/O‑bound → 20..200 (подбирать по метрикам)
- длинные задачи → часто prefetch=1 даёт стабильность

Сигнал, что prefetch слишком большой:

- `messages_unacknowledged` растёт
- потребление памяти на consumer’ах растёт
- долгие хвосты latency

### 3) Publisher confirms и batching

Publisher confirms дают гарантию, но добавляют overhead.

Best practice:

- включать confirms для важных потоков
- использовать batching/буферизацию на стороне producer’а (если библиотека поддерживает), чтобы уменьшить overhead на round-trips

### 4) Размер сообщений и компрессия

- большие payload’ы → нагрузка на сеть/диск/память
- часто лучше хранить payload в object storage/БД, а в RabbitMQ передавать ссылку + метаданные

---

## Lazy queues (идея)

Lazy‑поведение означает, что сообщения стараются держать на диске, а не в памяти, что полезно при больших backlog’ах.

Production смысл:

- если “нормально”, что очередь может разрастись (пики трафика), лучше контролировать память

Риск:

- диск становится bottleneck (latency растёт), особенно на медленных volumes

---

## Нагрузочное тестирование: как делать правильно

### 1) Что фиксировать до теста

- размер сообщения (bytes) и формат (json/protobuf)
- durability/persistence (durable queue, persistent messages)
- ack strategy и prefetch
- topology (classic vs quorum, число нод)

Иначе сравнивать результаты бессмысленно.

### 2) Сценарии

- увеличить producers → смотреть publish rate и latency
- увеличить consumers → смотреть ack rate и backlog
- намеренно “сломать consumer” → проверить DLQ/retry и устойчивость

### 3) Где bottleneck чаще всего

- disk I/O (особенно quorum)
- сеть между нодами/клиентами
- слишком много соединений/каналов
- consumer‑логика (БД/HTTP зависимости)

---

## Production best practices (performance)

- **Держать connections под контролем**: connection pooling на уровне приложения (1 conn / process), channels per worker.
- **Prefetch всегда настраивать**: по умолчанию легко получить перекос и memory‑пики.
- **Не передавать большие payload’ы** через брокер без необходимости: ссылки вместо данных.
- **Выделять отдельные очереди по профилю нагрузки** (короткие/длинные задачи), чтобы длинные задачи не ухудшали latency коротких.
- **Мониторить in-flight/backlog/rates** и связывать с изменениями конфигурации.

---

## Дополнительные материалы

- [RabbitMQ — Performance](https://www.rabbitmq.com/performance.html)
- [RabbitMQ — Consumer Prefetch](https://www.rabbitmq.com/consumer-prefetch.html)
