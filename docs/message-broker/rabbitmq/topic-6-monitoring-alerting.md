# 6. Мониторинг и алертинг

---
Цель темы — видеть проблемы RabbitMQ **до** падения системы: рост очередей (backlog), перегруз consumer’ов (lag/in‑flight), memory/disk alarms, падение нод, деградация publish/ack rate. Здесь — что мониторить, как подключить exporter, какие графики нужны и какие алерты реально полезны в production.

---

## Что мониторить (минимальный набор)

### Очереди

- **Queue depth / backlog**: `messages_ready` — сколько сообщений ждёт обработки.
- **In-flight**: `messages_unacknowledged` — сколько сообщений у consumer’ов «в работе» без ack.
- **Consumers**: `consumers` — сколько consumer’ов подписано на очередь.

Практическая интерпретация:

- растёт `messages_ready` → система **не успевает** (мало consumer’ов, медленная обработка, ошибки)
- растёт `messages_unacknowledged` → consumer’ы **берут слишком много** (prefetch высок), или обработка подвисает
- `consumers = 0` при не пустой очереди → outage цепочки обработки

### Ноды и ресурсы

- **Memory alarm** / **Disk alarm** — сигнал backpressure (publish замедлится, возможны ошибки).
- Использование диска и I/O latency (особенно важно для quorum очередей).

### Скорости

- publish rate / deliver rate / ack rate — помогает отличить “просто пик” от “система деградирует”.

---

## Инструменты

| Инструмент | Роль |
|-----------|------|
| **RabbitMQ Management UI** | Быстрая визуальная диагностика очередей/соединений/каналов; удобно для ручного расследования. |
| **Prometheus** | Сбор метрик и алертинг (часто через Alertmanager). |
| **Grafana** | Дашборды (queue depth, rates, alarms, per‑vhost/per‑queue). |

---

## Подключение метрик (варианты)

### Вариант 1: встроенный Prometheus endpoint (плагин)

RabbitMQ умеет отдавать метрики для Prometheus через plugin `rabbitmq_prometheus` (в зависимости от дистрибутива/чарта может быть включено).

В Kubernetes чаще всего это уже предусмотрено chart/operator’ом:

- создаётся Service на порт `/metrics`
- Prometheus скрейпит его через ServiceMonitor/PodMonitor (если используется operator)

### Вариант 2: exporter рядом

Иногда ставят отдельный exporter (реже в современных установках). Принцип тот же: Service + scrape.

---

## Пример (Kubernetes): ServiceMonitor для метрик RabbitMQ

```yaml
# Пример для Prometheus Operator.
# Важно: selector и port должны совпадать с Service, который экспонирует /metrics.
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rabbitmq
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames: ["rabbitmq"]
  selector:
    matchLabels:
      app.kubernetes.io/name: rabbitmq
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

Комментарий: это “скелет”. В реальном кластере метки и имя порта зависят от чарта/operator’а.

---

## Дашборды: какие графики нужны

### 1) Backlog и обработка

- `messages_ready` (по очередям и суммарно по vhost)
- `messages_unacknowledged`
- consumer count
- publish/deliver/ack rate

### 2) Alarms и ресурсы

- memory alarm/disk alarm (флаг и длительность)
- disk free и I/O latency

### 3) Соединения и каналы

- connections/channels — резкие скачки часто признак неправильного использования (connect per message)

---

## Production алерты (практичные)

Ниже — логика алертов (без привязки к конкретным именам метрик — они могут отличаться по exporter’у).

### Queue растёт N минут

- Условие: `messages_ready` растёт или превышает порог и держится > 10–15 минут.
- Смысл: consumer’ы не успевают или не работают.

### Нет consumer’ов

- Условие: `consumers == 0` и `messages_ready > 0` (или rate publish > 0).
- Смысл: “сервис обработки умер”.

### Много unacked

- Условие: `messages_unacknowledged` высокое и растёт.
- Смысл: prefetch слишком большой, обработка зависла, или ack’и не приходят.

### Memory/Disk alarm

- Условие: alarm активен > 1–5 минут.
- Смысл: broker включает backpressure; дальше будет деградация и ошибки.

### Нода кластера down

- Условие: количество running нод < ожидаемого.
- Смысл: риск потери кворума (особенно для quorum очередей) и деградации.

---

## CLI‑диагностика (быстро)

```bash
# Очереди: backlog / in-flight / consumers
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers durable

# Соединения и каналы (часто видно «утечки»)
rabbitmqctl list_connections name user peer_host peer_port state channels
rabbitmqctl list_channels connection pid number consumer_count messages_unacknowledged
```

---

## Небольшой пример кода: правильное использование connection/channel

Антипаттерн: создавать connection на каждое сообщение.

Правильно: держать 1 connection и переиспользовать channel.

```python
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()

for i in range(1000):
    ch.basic_publish(exchange="", routing_key="tasks", body=f"job-{i}".encode("utf-8"))

conn.close()
```

Production best practice: добавлять publisher confirms и backoff при ошибках публикации (см. тему 2).

---

## Дополнительные материалы

- [RabbitMQ — Monitoring](https://www.rabbitmq.com/monitoring.html)
- [RabbitMQ — Prometheus](https://www.rabbitmq.com/prometheus.html)
