# 1. База: архитектура и первая практика

---

В этой странице — фундамент Kafka для production-понимания: как устроены broker/кластер, зачем нужны partition’ы, что означает replication и ISR, как работают consumer groups и offset’ы, и как поднять Kafka локально в двух режимах (ZooKeeper и KRaft).

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Broker** | Узел Kafka, который принимает и обслуживает данные/запросы. В production обычно несколько broker’ов. |
| **Cluster** | Набор broker’ов, работающих вместе как единая система. |
| **Раздел Kafka** | Именованная логическая сущность, куда публикуются сообщения; физически данные распределяются по partition’ам. |
| **Partition** | Фрагмент данных раздела Kafka; partition — единица параллелизма и масштабирования. |
| **Producer** | Клиент, который публикует сообщения в Kafka. |
| **Consumer** | Клиент, который читает сообщения из Kafka. |
| **Replication** | Репликация partition’а по нескольким broker’ам для отказоустойчивости. |
| **Leader / Follower** | Для каждого partition существует leader (принимает запись/читает “как основная точка”) и followers (реплики данных). |
| **ISR (in-sync replicas)** | Реплики, которые синхронизированы с leader’ом. ISR определяет, можно ли “продолжать” запись при отказах. |
| **Consumer group** | Логическая группа consumer’ов для параллельной обработки: partition’ы распределяются между членами группы. |
| **Offset** | Позиция в partition’е, указывающая, до каких записей consumer уже дошёл. |

---

## Архитектура потока данных (что происходит при записи/чтении)

Примерная логика:

`producer -> раздел Kafka -> partition -> (replication) -> consumer group -> offset`

Ключевые идеи:

- запись идёт в partition’ы (а не “в одно место”),
- внутри partition есть leader и репликация на followers,
- consumer group читает partition’ы параллельно, а offset’ы “фиксируют прогресс”.

---

## Partition: почему это важно для DevOps

Partition’ы — это:

- **масштабирование параллелизма**: больше partition’ов = потенциально больше consumer-ов в группе,
- **границы доставки**: порядок гарантируется внутри partition, но не между partition’ами,
- **единица репликации**: replication factor влияет именно на partition.

Production-практики:

- заранее планируйте число partition’ов (изменение после запуска дороже и не всегда прямое),
- держите разумный баланс: “слишком мало” partition’ов упирается в throughput, “слишком много” увеличивает накладные расходы (файлы/метаданные/rebalance).

---

## Replication, ISR, отказоустойчивость

Что означает replication factor:

- определяет, сколько broker’ов будут хранить реплики partition’а,
- задаёт “резерв” на случай недоступности части broker’ов.

ISR — это “реально синхронизированные” реплики:

- пока лидер может продолжать запись и поддерживать ISR, кластер устойчив,
- при падениях может измениться лидер/ISR состав, и это отражается на времени восстановления.

Best practice:

- для production выбирайте replication factor с учётом количества broker’ов и отказов (обычно минимум 3 broker’а для rf=3),
- не забывайте про storage и network: репликация упирается в throughput/latency дисков и сети.

---

## Consumer groups и offset management (как не “сломать” доставку)

Consumer group распределяет partition’ы между членами группы:

- если в группе 3 consumer’а и 9 partition’ов — на каждого в среднем приходится по 3 partition’а,
- если добавить/убрать consumer’ов — происходит rebalance и перераспределение partition’ов.

Offset’ы бывают:

- **автоматически** управляемые (риски при ошибках обработки),
- **ручные commit** (часто лучше для idempotency и correct-at-least-once).

Production best practices:

- commits делайте только после успешной обработки (или используйте транзакционный подход),
- планируйте повторную обработку как норму: “at-least-once” может доставлять дубликаты,
- добавляйте idempotency ключи в обработку, если бизнесу критичны повторные события.

---

## Практика 1: поднять Kafka локально (Docker Compose)

Ниже два варианта — ZooKeeper mode и KRaft mode. Для удобства обе конфигурации рассчитаны на dev:

- включено автосоздание разделов (чтобы быстро получить partition’ы по умолчанию),
- заданы defaults для числа partition’ов и replication factor.

### Вариант A: ZooKeeper mode (dev)

```yaml
version: "3.9"

services:
  zookeeper:
    image: bitnami/zookeeper:3.9
    ports:
      - "2181:2181"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes

  kafka:
    image: bitnami/kafka:3.7
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      # ZooKeeper адрес для кластера
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181

      # Локальный адвертайзинг (важно для клиента с хоста dev-машины)
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - KAFKA_CFG_LISTENERS=PLAINTEXT://0.0.0.0:9092

      # Упрощаем dev: разделы создаются автоматически при первой записи
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true

      # Defaults для автосоздания (чтобы разделы появлялись сразу с несколькими partition’ами)
      - KAFKA_CFG_NUM_PARTITIONS=3
      - KAFKA_CFG_DEFAULT_REPLICATION_FACTOR=1
```

### Вариант B: KRaft mode (современный dev)

```yaml
version: "3.9"

services:
  kafka:
    image: bitnami/kafka:3.7
    ports:
      - "9092:9092"
    environment:
      # В KRaft роль брокера/контроллера задаётся через process roles
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER

      # Listener’ы: client traffic и controller traffic
      - KAFKA_CFG_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092

      # Dev defaults: автосоздание и количество partition’ов
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true
      - KAFKA_CFG_NUM_PARTITIONS=3
      - KAFKA_CFG_DEFAULT_REPLICATION_FACTOR=1
```

---

## Практика 2: создать раздел с несколькими partition’ами (быстро, dev-путь)

В dev-подходе выше partition’ы появляются автоматически благодаря defaults:

- `KAFKA_CFG_NUM_PARTITIONS=3`

То есть достаточно произвести первое сообщение в новый раздел — он будет создан Kafka’ой с нужным количеством partition’ов.

---

## Практика 3: producer/consumer (CLI, быстро проверить pipeline)

Для экспериментов удобно использовать `kcat` (бывший kafkacat).

### Producer: отправить несколько сообщений

```bash
kcat -b localhost:9092 \
  -t orders-events \
  -P \
  -K: -l

# Идея: строка “value” читается из stdin.
# Пример: запустите и по очереди введите строки, затем Ctrl+D.
```

### Consumer: прочитать сообщения из раздела

```bash
kcat -b localhost:9092 \
  -t orders-events \
  -C \
  -o beginning \
  -f '%o %t %s\n'
```

Production-подсказка:

- в dev смотрим “начало” (`-o beginning`),
- в production consumer’ы обычно стартуют с сохранённых offset’ов и используют ручной commit/семантику обработки.

---

## Что важно для production (короткая checklist)

1) **Partition plan заранее**: продумывайте количество partition’ов под throughput и количество consumer’ов.
2) **Replication factor**: выбирайте rf под реальное число broker’ов и ожидаемые отказы.
3) **Idempotency и commit strategy**: коммитьте offset’ы после успешной обработки и проектируйте повторные доставки.
4) **Отключайте auto-create в прод**: разделы должны появляться управляемо через инфраструктурные процессы.

