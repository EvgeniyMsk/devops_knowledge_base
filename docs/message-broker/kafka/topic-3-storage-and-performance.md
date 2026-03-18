# 3. Хранение и производительность

---

Kafka — это distributed log: то, как вы храните данные на диске, напрямую влияет на задержки (latency), стоимость операций и предсказуемость поведения при нагрузках и отказах.

В этой теме разбираем:

- log-based storage и сегменты,
- retention по времени и по размеру,
- compaction для “последнего состояния” по ключу,
- как проверить, что удаление/компакция работают как задумано,
- DevOps-фокус: disk I/O bottlenecks, page cache и JBOD vs RAID.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Log (журнал)** | Накопление записей, разложенное на сегменты. |
| **Segment** | Физический фрагмент журнала, который Kafka периодически создаёт и обслуживает. |
| **Retention** | Политика удаления “устаревших” сегментов/записей. |
| **Time-based retention** | Удаление по времени (`retention.ms`). |
| **Size-based retention** | Удаление по достижению лимита (`retention.bytes`). |
| **Compaction** | Сжатие лога по ключу: сохраняет “последнее значение” для каждого ключа (логика log compaction). |
| **Tombstone** | Запись с ключом и “пустым значением” (обычно помогает compaction удалять ключ из результата). |
| **Cleaner (log cleaner)** | Компонент Kafka, который выполняет compaction для логов. |
| **JBOD** | Архитектура хранения, где диски используются “как есть” (без RAID-мультипликации). |
| **RAID** | Избыточные массивы дисков для отказоустойчивости и/или производительности. |

---

## Log-based storage: сегменты и “почему delete не мгновенный”

Kafka физически пишет данные в сегменты. Политики retention/compaction работают на уровне сегментов и фоновых процессов:

- при time-based retention запись может быть “старой”, но удаление сегмента произойдёт после выполнения логики очистки,
- компакция также выполняется асинхронно cleaner’ом,
- на практике “видимое удаление/сжатие” обычно имеет задержку до next cleanup cycle.

Production best practices:

- планируйте retention как SLA на уровне “сколько времени данные доступны для чтения”, а не как “мгновенно исчезают”,
- контролируйте фоновые очистки (cleaner throughput, задержки), иначе retention начнёт отставать.

---

## Retention policies: время и размер

### Time-based retention (`retention.ms`)

Смысл: держим записи в логе не дольше заданного времени.

Чем полезно:

- предсказуемость “окна” данных,
- удобные требования для compliance/data retention.

Типовой production пример (Strimzi):

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: orders-events
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 6
  replicas: 3
  config:
    retention.ms: 604800000 # 7 дней
    segment.bytes: 1073741824 # размер сегмента (1GiB)
```

### Size-based retention (`retention.bytes`)

Смысл: держим суммарный объём данных не выше заданного.

Плюсы:

- стабильные требования по диску,
- хорошо подходит для потоков с “непредсказуемым” количеством событий.

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: clickstream
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 8
  replicas: 3
  config:
    retention.bytes: 107374182400 # 100GiB на partition’а (учитывайте реальную модель)
```

Production best practices:

- используйте либо time-based, либо size-based (или осознанно комбинируйте), чтобы понимать, что именно “упирает” диски,
- корректно подбирайте `segment.bytes`: слишком большие сегменты могут задерживать очистку,
- всегда соотносите retention с размером диска, IOPS и планом роста.

---

## Compaction: когда “последнее значение по ключу” важнее истории

Compaction полезен, когда вы хотите хранить агрегированное/текущее состояние по ключу (например, last-known state), а историю “с тем же ключом” можно вытеснять.

Условие: компакция работает по ключам. Если ключ не задан (или ключ уникален почти всегда), результат будет менее полезен.

### Включение compaction (`cleanup.policy=compact`)

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: customer-state
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 6
  replicas: 3
  config:
    cleanup.policy: compact
    # сегментация влияет на эффективность очистки
    segment.bytes: 1073741824
```

Комментарий:

- `cleanup.policy=compact` включает log compaction,
- tombstone помогает “удалить ключ” из результата для компакции.

### Комбинированная политика (compact + delete)

Иногда нужен компромисс: компакция по ключу + удаление слишком старых данных.

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: customer-state-retained
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 6
  replicas: 3
  config:
    cleanup.policy: compact,delete
    retention.ms: 259200000 # 3 дня на уровне delete
```

Production best practices:

- заранее определяйте “что является источником истины”: история или текущее состояние,
- если используете compaction — убедитесь, что продакшен-код действительно публикует стабильные ключи,
- тестируйте tombstone поведение end-to-end (producer, serializers, consumer semantics).

---

## Как проверить, что retention/compaction работают

### Практика проверки retention (идея)

1) создайте раздел с коротким retention (в dev/test),
2) отправьте несколько сообщений,
3) подождите период > retention,
4) убедитесь, что данные больше не читаются (или читаются только в пределах ожидаемого окна).

Пример команды чтения в стиле “всё из начала” (kcat):

```bash
kcat -b kafka:9092 \
  -t orders-events \
  -C -o beginning
```

### Практика проверки compaction: “один ключ -> последнее значение”

Идея:

- публикуем записи с одним и тем же ключом и меняющимся значением,
- читаем лог и видим, что остаётся последнее значение для ключа (плюс поведение зависит от timing cleaner’а).

Пример producer/consumer с ключами (kcat):

```bash
# Producer (ключ и значение). Порядок сообщений важен.
kcat -b kafka:9092 -t customer-state -P -K: \
  -l <<'EOF'
cust-1:value-1
cust-1:value-2
cust-1:value-3
EOF

# Consumer: читаем “как есть” из начала
kcat -b kafka:9092 -t customer-state -C -o beginning
```

Tombstone (ключ + пустое значение) — логика зависит от serializer’ов, но concept такой:

```bash
kcat -b kafka:9092 -t customer-state -P -K:
  # отправьте запись с тем же ключом и пустым value
```

Production best practices:

- для честной проверки compaction учитывайте “грязность” (dirty segments) и throughput cleaner’а,
- смотрите broker metrics по cleaner’у (если у вас есть мониторинг).

---

## DevOps-фокус: disk I/O bottlenecks, page cache, JBOD vs RAID

### Disk I/O bottlenecks

Симптомы:

- producer начинает замедляться (acks растут),
- consumer latency увеличивается,
- очистки retention/compaction начинают “отставать”.

Причины:

- слишком медленный storage / сеть,
- неверные request/limit для broker’ов,
- слишком активные background операции (cleaner),
- конкуренция за диски между broker’ами на одном узле.

Best practices:

- выделяйте storage performance классы под broker’ы,
- ограничивайте плотность broker’ов на узел,
- наблюдайте p99 latency диска и сеть.

### Page cache (OS cache)

Kafka часто упирается не только в raw disk, но и в page cache:

- при “горячих” чтениях данные могут попадать в RAM и быть быстрее диска,
- при недостатке памяти cache evict’ится, и latency резко растёт.

Best practices:

- выделяйте достаточно RAM broker’ам (не только CPU),
- избегайте агрессивного memory overcommit на узлах кластера,
- если есть separate pages cache tuning в кластере — делайте тесты на нагрузке.

### JBOD vs RAID

Практически в production чаще предпочитают JBOD, потому что:

- вы получаете более предсказуемое распределение нагрузки по дискам,
- меньше “задержек” и сложностей, связанных с полосами/паритетом RAID,
- проще масштабировать диски/узлы.

RAID иногда выбирают для соответствия корпоративным требованиям по отказоустойчивости на уровне дисков, но:

- RAID-контроллер может менять профиль latency (особенно при rebuild/parity operations),
- при некоторых паттернах IO можно получить более высокий tail latency.

Best practices:

- делайте выбор JBOD/RAID на основе профиля нагрузки (write/read mix),
- проводите тестирование на representative dataset и failure scenarios,
- планируйте рост storage как “часть архитектуры”, а не как “ручное расширение” в последний момент.

---

## Production checklist (коротко)

| Проверка | Зачем |
|----------|-------|
| retention/compaction выбран осознанно | Чтобы cost и поведение соответствовали SLA |
| segment.bytes подобран под очистку | Чтобы delete/cleaner не отставал |
| есть end-to-end тест компакции/тombstone | Чтобы consumer semantics были корректны |
| мониторинг cleaner/latency/disc IO | Чтобы видеть проблемы раньше incident |
| storage архитектура продумана (JBOD/RAID) | Чтобы tail latency оставался управляемым |

