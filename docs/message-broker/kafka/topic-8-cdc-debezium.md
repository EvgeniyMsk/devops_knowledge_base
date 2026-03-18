# 8. CDC и Debezium

---

CDC (Change Data Capture) превращает изменения из OLTP в поток событий для дальнейшей обработки.

Debezium — популярный CDC-инструмент в связке с Kafka Connect: он читает изменения БД из логических журналов и публикует события в Kafka через Connect.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **CDC** | Change Data Capture: доставка изменений данных (INSERT/UPDATE/DELETE) как событий. |
| **Logical replication (PostgreSQL)** | Механизм публикации изменений на уровне логических событий (через replication slot’ы). |
| **binlog (MySQL)** | Журнал изменений MySQL, который Debezium может читать для CDC. |
| **Debezium connector** | Конфигурация Debezium, подключающаяся к БД и публикующая события в Kafka. |
| **Schema evolution** | Изменение схемы данных (DDL) со временем: совместимость сообщений и версий схем. |
| **Backpressure** | Ситуация, когда downstream не успевает: накапливаются задержки, растёт consumption ресурсов. |
| **Schema Registry** | Компонент (обычно Confluent), который хранит и версионирует схемы (Avro/Protobuf/JSON Schema). |

---

## Архитектура CDC в “production-логике”

Упрощённая последовательность:

1) включаем логическое чтение изменений в БД,
2) Debezium connector подписывается на изменения,
3) Kafka Connect доставляет события в Kafka,
4) consumer’ы применяют бизнес-логику,
5) изменения схемы (DDL) совместимы и версионированы.

Production best practice:

- планируйте совместимость схем заранее: DDL в OLTP должен быть “совместим с пайплайном”.

---

## Практика: включить logical replication в PostgreSQL

Точные шаги зависят от вашей версии и политики безопасности, но производственная логика обычно такая:

```sql
-- 1) Включите параметры в postgresql.conf (понадобится restart):
-- wal_level = logical
-- max_replication_slots = 10
-- max_wal_senders = 10

-- 2) Role для репликации
CREATE ROLE debezium_replication WITH REPLICATION LOGIN PASSWORD 'REPLACE_ME';

-- 3) Publication: какие таблицы/набор событий доступны для логической репликации
CREATE PUBLICATION dbserver_pub FOR ALL TABLES;
```

Комментарий:

- replication slot’ы удерживают WAL, пока прогресс connector’а не продвинется,
- поэтому monitoring задержки CDC критичен: он напрямую влияет на размер WAL и стабильность Postgres.

---

## Практика: создать Debezium connector через REST API Kafka Connect

Ниже пример POST-запроса на создание connector’а. Это production-minded скелет: параметры источника, publication, режим snapshot и параллелизм.

Поля backend для истории схем зависят от вашей версии Debezium/Connect — ниже они опущены, но должны быть настроены по вашему стандарту.

```bash
curl -X POST http://kafka-connect:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "debezium-postgres",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "postgres",
      "database.port": "5432",
      "database.user": "debezium_replication",
      "database.dbname": "appdb",

      "plugin.name": "pgoutput",
      "publication.name": "dbserver_pub",

      "snapshot.mode": "initial",
      "tasks.max": "2"
    }
  }'
```

Production best practices:

- snapshot планируйте в окна с контролируемой нагрузкой на OLTP,
- задачи/параллелизм (`tasks.max`) выбирайте под throughput downstream, иначе lag начнёт расти,
- после DDL — проверьте совместимость схем на consumer’ах.

---

## Schema evolution: совместимость, а не “угадывание”

Когда БД меняет схему:

- могут появляться новые поля,
- меняются типы,
- добавляются/убираются таблицы и ключи.

Production best practices:

- используйте стратегию совместимости схем (backward/forward) и версионируйте сообщения,
- применяйте Schema Registry как единый источник схем,
- обновляйте consumer’ы и schema compatibility в согласованном порядке.

Мини-практика: проверить совместимость (concept)

```bash
curl -X POST http://schema-registry:8081/compatibility/subjects/<subject>/versions/latest
```

---

## Exactly-once vs at-least-once в CDC

В CDC чаще всего реальность такая:

- доставка в Kafka — ближе к at-least-once (дубликаты возможны),
- а “exactly-once” на уровне бизнеса достигается идемпотентностью обработки.

Production best practices:

- dedup по стабильному ключу события (например, LSN/txid + идентификатор строки/изменения),
- commit progress делайте только после успешного apply_change,
- для критичных интеграций используйте outbox-паттерн на стороне приложения (если применимо).

---

## Backpressure и lag CDC: как реагировать

Если лаг растёт:

- проверьте downstream (DB/HTTP) на slow paths,
- увеличьте параллелизм consumer’ов/worker’ов (не только в Kafka),
- убедитесь, что connector жив и replication slot’ы не “застряли”,
- проверьте совместимость схем (DDL мог сломать consumer’а).

---

## Мини-код: идемпотентная обработка событий (Python-псевдокод)

Цель — не применять одно и то же событие дважды.

```python
processed_event_ids = set()

for msg in consumer:
    event_id = msg.headers.get("event-id")  # пример: идемпотентный ключ события
    if event_id in processed_event_ids:
        continue

    apply_change(msg.value())
    processed_event_ids.add(event_id)

    # commit делайте после успешного apply_change
    consumer.commit()
```

Комментарий:

- в production дедуп ключи лучше хранить в durable store (БД/Redis с TTL), а не в памяти,
- commit/offset progress связывайте с успешностью apply_change.

---

## Production checklist

| Что проверить | Зачем |
|---------------|-------|
| logical replication включён и доступен Debezium | чтобы события вообще появлялись |
| Lag CDC не растёт устойчиво | чтобы не раздувать WAL и не накапливать очередь |
| Schema evolution совместима | чтобы DDL не ломал консьюм |
| Consumer обработка идемпотентная | чтобы дубликаты не ломали бизнес |
| Есть план реакции на backpressure | чтобы MTTR был быстрым |

