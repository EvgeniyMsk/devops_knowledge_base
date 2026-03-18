# 8. Troubleshooting

---
Это практическая тема для инцидентов: как диагностировать, когда **сообщения “зависли”**, **очередь растёт**, **consumer’ы падают**, включились **memory/disk alarms**, или случился **network partition**. Фокус — быстрый triage, проверка гипотез и минимальный набор команд/метрик.

---

## Быстрый triage (5–10 минут)

1) **Что именно болит?**  
Очередь растёт (backlog), задержки выросли, появились ошибки publish/consume, падают consumer’ы?

2) **Где узкое место?**  
Broker (alarms, диск), consumer’ы (ошибки, медленные), сеть (partition/latency), producer (шторм/ретраи)?

3) **Какая очередь/маршрут?**  
В RabbitMQ почти всё решается на уровне конкретных очередей/exchange/vhost.

---

## Симптом: очередь растёт (messages_ready ↑)

### Частые причины

- consumer’ов меньше, чем нужно (или они “умерли”)
- consumer’ы работают медленнее входящего потока (БД/HTTP зависимость, throttling)
- retry‑петля (сообщения возвращаются и снова попадают в очередь)

### Проверки (CLI)

```bash
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers durable
```

Интерпретация:

- `consumers = 0` → outage consumer’ов (смотреть деплой/процессы/коннекты)
- `messages_unacknowledged` высокое → consumer’ы “взяли в работу”, но не ack (зависли/падают)

### Действия

- временно **масштабировать consumer’ов** (если безопасно)
- проверить **prefetch** (слишком высокий → много in‑flight и memory‑пики)
- убедиться, что есть **DLQ** и ретраи ограничены (см. тему 2)

---

## Симптом: messages_unacknowledged ↑ (in-flight растёт)

### Причины

- consumer завис (I/O, блокировки, таймауты)
- слишком высокий prefetch
- consumer падает до ack (CrashLoop)

### Проверки

```bash
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
rabbitmqctl list_channels connection pid number consumer_count messages_unacknowledged
```

### Действия

- уменьшить prefetch
- добавить timeouts в consumer (БД/HTTP)
- проверять, что ack делается **после** фиксации результата (иначе будет “потеря” работы)

---

## Симптом: consumer’ы падают

### Проверки

- логи consumer‑приложения (исключения, timeouts, OOM)
- количество соединений/каналов (нет ли “штормов”)

RabbitMQ‑сторона:

```bash
rabbitmqctl list_connections name user peer_host peer_port state channels
```

### Production best practices

- consumer должен быть **идемпотентным** (повторы неизбежны)
- ошибки классифицировать:
  - временные → retry с задержкой (TTL/DLX)
  - постоянные (bad payload) → DLQ, без бесконечного requeue

---

## Симптом: memory alarm

Когда включается memory alarm, RabbitMQ активирует backpressure: producers начинают замедляться/получать ошибки, latency растёт.

### Проверки

- memory alarm активен? (смотреть в Management UI или метрики)
- растёт ли `messages_unacknowledged` (часто из-за prefetch)

### Действия

- снизить prefetch / ограничить параллелизм consumer’ов
- проверить размер сообщений (слишком большие payload’ы)
- убедиться, что Kubernetes limits не слишком малы (иначе OOMKilled и flapping)

---

## Симптом: disk alarm / заканчивается место

Это критично: диск нужен для метаданных и (часто) сообщений. Disk alarm также приводит к блокировкам публикации.

### Проверки

- disk free на ноде/volume
- быстрый рост backlog (сообщения складываются на диск)

### Действия

- увеличить volume / расширить PVC (если поддерживается)
- включить/проверить ретеншн‑политику и TTL для “временных” сообщений
- убедиться, что DLQ не растёт бесконтрольно (и есть процесс разборки DLQ)

---

## Симптом: network partition / split cluster

### Признаки

- часть нод не видит другие
- ошибки соединения/кластерной синхронизации
- деградация quorum очередей (потеря кворума)

### Проверки

```bash
rabbitmqctl cluster_status
rabbitmqctl status
```

### Действия

- стабилизировать сеть (это первопричина)
- не делать “ручных чудес”, пока не понятно, какая часть кластера в большинстве/меньшинстве
- иметь заранее выбранную стратегию partition handling и протестированный план восстановления (см. тему 4)

---

## Практика: симуляции (на стенде)

1) **Падение consumer**: остановить consumer’ы → увидеть рост `messages_ready` → проверить алерты.  
2) **Переполнение очереди**: запустить producer‑шторм → смотреть disk/memory alarms.  
3) **Network partition**: “разрезать” сеть между нодами → посмотреть cluster_status и поведение quorum очередей.

Цель — чтобы во время реального инцидента вы действовали по знакомому сценарию.

---

## Полезные ссылки

- [RabbitMQ — Monitoring](https://www.rabbitmq.com/monitoring.html)
- [RabbitMQ — Troubleshooting](https://www.rabbitmq.com/troubleshooting.html)
- [RabbitMQ — Network Partitions](https://www.rabbitmq.com/partitions.html)
