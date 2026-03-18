# 7. Kafka Connect

---

Kafka Connect используется, чтобы надёжно подключать внешние системы к Kafka и обратно через connector’ы.

В production Connect обычно:

- работает в распределённом режиме (Distributed),
- масштабируется worker’ами/тасками,
- имеет чёткую схему fault tolerance,
- и управляется конфигурацией как код (и секретами отдельно).

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Kafka Connect** | Платформа для интеграции: запускает connector’ы, которые читают/пишут данные в Kafka. |
| **Connector** | Конфигурация и логика, которая создаёт задачи для чтения/записи (source или sink). |
| **Source connector** | Забирает данные из внешнего источника и публикует их в Kafka. |
| **Sink connector** | Читает данные из Kafka и пишет во внешнюю систему. |
| **Worker** | Pod/процесс Connect, который управляет execution’ом задач. |
| **Task** | Единица параллелизма внутри connector’а (может обрабатываться независимо). |
| **Standalone vs Distributed** | Способ запуска: Standalone для простых сценариев, Distributed для production-систем. |

---

## Source / Sink connector’ы: базовая модель

Почти всегда схема выглядит так:

`external system -> connector -> Kafka (для source)`

и наоборот:

`Kafka -> connector -> external system (для sink)`

Production best practices:

- выделяйте отдельные connector’ы под разные домены данных (меньше blast radius),
- планируйте параллелизм через number of tasks,
- проектируйте повторные попытки и идемпотентность на стороне принимающей системы.

---

## Distributed vs Standalone режимы

### Standalone (для dev/быстрого старта)

Плюсы:

- проще поднять,
- быстро проверить connector.

Минусы:

- ограниченная отказоустойчивость,
- сложнее масштабировать и управлять жизненным циклом в production.

### Distributed (для production)

Плюсы:

- управляемая отказоустойчивость,
- масштабирование worker’ами,
- легче централизованно управлять offset’ами и конфигурацией.

Best practice:

- используйте Distributed для всех систем, где Connect влияет на SLO.

---

## Практика: поднять Kafka Connect (концепт)

В production Connect чаще разворачивают в Kubernetes как Deployment (или StatefulSet при необходимости) с:

- отдельным namespace,
- persistent storage для Connect offsets/config (если требуется),
- корректной сетевой связностью с Kafka.

Мини-скелет (идея), без привязки к конкретному оператору:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-connect
spec:
  replicas: 2 # production: минимум 2 для устойчивости к рестартам
  template:
    spec:
      containers:
        - name: connect
          image: your-connect-image:latest
          ports:
            - containerPort: 8083 # REST API Connect
          env:
            - name: CONNECT_BOOTSTRAP_SERVERS
              value: "kafka-0.kafka:9092,kafka-1.kafka:9092"
```

Комментарий:

- реальная конфигурация (mode, converters, offset storage, status storage) задаётся файлами/конфига-провайдерами, а не только env.

---

## Примеры connector’ов (небольшие конфиги)

Ниже — минимальные примеры POST-запросов в REST API Connect для создания connector’ов.

### 1) File source connector (пример для чтения файлов)

```bash
curl -X POST http://kafka-connect:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "file-source-orders",
    "config": {
      "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
      "tasks.max": "2",
      "file": "/data/incoming/orders/",
      "destination.prefix": "orders-events-", 
      # В реальном конфиге у FileStreamSourceConnector есть параметр для префикса публикации
      "finished.path": "/data/processed/orders/",
      "ignore.utf8.errors": "false"
    }
  }'
```

Best practices:

- задавайте `finished.path` чтобы файлы не читались повторно бесконечно,
- ограничивайте параллелизм через `tasks.max` и следите за нагрузкой на broker/приёмники.

### 2) JDBC sink connector (пример записи в БД)

```bash
curl -X POST http://kafka-connect:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "jdbc-sink-orders",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      "tasks.max": "4",
      "connection.url": "jdbc:postgresql://postgres:5432/app",
      "auto.create": "false",
      "insert.mode": "insert",
      "table.name.format": "orders_events",
      "destination": "orders-events"
      # В реальном конфиге у JdbcSinkConnector есть поле(я) для выбора источников из Kafka
    }
  }'
```

Best practices:

- избегайте `auto.create=true` в production (схема БД должна управляться миграциями),
- включайте batch/timeout настройки под ваши SLA (не только базовые поля),
- защищайте креденшелы через secrets store.

---

## DevOps-фокус: scaling workers и fault tolerance

### Scaling workers

При росте нагрузки:

- увеличивайте количество worker’ов в Distributed mode,
- увеличивайте `tasks.max` (но следите за downstream system’ами),
- следите за offset lag/processing lag через мониторинг Kafka.

Best practices:

- масштабируйте по очереди: сначала worker’ы/таски, потом тюните конвертеры/батчинг,
- делайте нагрузочное тестирование “с реальным downstream”.

### Fault tolerance

В production сценариях Connect должен уметь переживать:

- рестарты worker’ов,
- временную недоступность внешних систем,
- ошибки парсинга/валидации.

Практика:

- для sink’ов используйте retries/backoff (и dead-letter strategy при необходимости),
- разделяйте connector’ы по типам данных, чтобы сбой одного не останавливал всё.

---

## Config management: как держать конфигурацию управляемой

Production approach:

- храните connector configs версионированно (Git),
- шаблонизируйте (Helm/Kustomize/Terraform) только то, что действительно меняется по окружениям,
- секреты (DB password, TLS certs) держите вне Git и прокидывайте как Secret/env/volume.

Комментарий:

- если вы обновляете connector config, делайте это как контролируемый rollout: проверка → подтверждение → закрепление.

---

## Production checklist

| Что проверить | Зачем |
|--------------|-------|
| Distributed режим для production | отказоустойчивость и масштабирование |
| `tasks.max` соответствует downstream throughput | чтобы не “задушить” внешнюю систему |
| Retry/timeout и DLQ-план для ошибок | меньше инцидентов “из-за одного bad message” |
| Offsets/config/status хранятся надёжно | чтобы worker restart не ломал прогресс |
| Секреты вынесены из Git и ротируются | безопасность и управляемость |

