# 6. Troubleshooting и диагностика

---
На **middle+** уровне от инженера ждут не только «кластер поднят», а умение **локализовать** деградацию: **лаг репликации**, **медленные запросы**, конкуренция за ресурсы, **давление на диск**. Ниже — типовые симптомы, инструменты (**mongostat**, **mongotop**, **profiler**, **логи**) и короткие команды; в конце — лабораторная идея с искусственной нагрузкой.

---

## Типовые проблемы

### Replication lag

| Симптом | Что проверить |
|---------|----------------|
| Вторичные **отстают** от primary | Сеть между узлами, диск/IOPS на secondary, тяжёлые записи, размер **oplog** |
| Чтения с secondary «старые» | Ожидаемо при лаге; `readConcern` / перенос чтения на primary для критичных путей |

```javascript
// Оценка состояния RS и задержек (mongosh)
rs.status()

// Кратко по членам: optimeDate, stateStr
rs.printSecondaryReplicationInfo()
```

**Production:** не маскируйте лаг увеличением **оплога** бесконечно — ищите **причину** (нагрузка, железо, антипаттерн запросов).

### Slow queries

- Долгие **`find`/`aggregate`** без подходящего индекса → **COLLSCAN**, рост CPU и диска.
- Неподходящий **sort** без индекса → in-memory sort / отказ при превышении лимитов.

```javascript
// План выполнения (всегда держите в арсенале)
db.orders.find({ userId: "u1", status: "open" })
  .sort({ createdAt: -1 })
  .explain("executionStats")

// Смотрите winningPlan.stage, totalDocsExamined vs totalDocsReturned
```

### «Lock contention» и конкуренция (WiredTiger)

В современных версиях глобальных **read/write lock** как в MMAPv1 нет; узкие места чаще выглядят так:

- **Высокая конкуренция** за одни и те же документы (много `update` одного `_id`).
- **Давление на cache** WiredTiger, **eviction**, очереди — рост latency при том же QPS.
- Долгие операции в **`currentOp`** (агрегации, массовые обновления).

```javascript
// Активные/долгие операции (осторожно: нагрузка на mongod при частом опросе)
db.currentOp({
  active: true,
  secs_running: { $gt: 5 },
})

// При необходимости — точечный kill (только по runbook и пониманию последствий)
// db.killOp(<opid>)
```

**Best practice:** для массовых мутирующих задач — **батчи**, **офф-пик**, индексы под фильтр; избегать «гонки» за один горячий документ без очередей/шардирования по ключу.

### Disk pressure

- Заполнение тома → риск остановки; медленный диск → рост latency и **лаг**.
- Рост данных без **архивации**/TTL там, где уместно.

```javascript
// Статистика по БД (размеры, индексы)
db.stats()
db.orders.stats()
```

---

## Инструменты: mongostat, mongotop, profiler, логи

### mongostat

Сводка **opcounters**, **connections**, **dirty** cache (в зависимости от версии) в консоли — быстрый «пульс» инстанса.

```bash
# К replica set через URI (утилиты из пакета database-tools / клиента MongoDB)
mongostat --uri="mongodb://user:pass@host:27017/?authSource=admin" 2
# аргумент 2 — интервал в секундах между строками
```

### mongotop

Время чтения/записи **по коллекциям** — помогает найти «горячие» коллекции.

```bash
mongotop --uri="mongodb://user:pass@host:27017/?authSource=admin" 5
```

### Profiler

Пишет медленные операции в коллекцию **`system.profile`** (учитывайте объём и **оверhead** на production).

```javascript
// Включить только slow-операции (порог в миллисекундах)
db.setProfilingLevel(1, { slowms: 100 })

// Или полный профиль (дорого — кратко и на stage)
// db.setProfilingLevel(2)

// Примеры чтения профиля
db.system.profile.find().sort({ ts: -1 }).limit(5)

// Выключить
db.setProfilingLevel(0)
```

**Production:** обычно **уровень 1** с адекватным `slowms`, **короткие** окна включения, мониторинг размера профиля; на постоянной основе иногда достаточно **логов** + **опциональный profiler** в stage.

### Логи

- Медленные запросы при **`slowOpThresholdMs`** в конфиге / **setParameter**.
- Ротация логов, централизованный сбор (ELK, Loki).

```javascript
// Порог медленной операции (мс) на живом процессе; осторожно в пике
db.adminCommand({ setParameter: 1, slowms: 200 })
```

---

## Практика: искусственная нагрузка и поиск узких мест

1. **Нагрузка:** скрипт или **mongo shell** циклом вставляет документы и выполняет **`find` без индекса** по полю с высокой селективностью там, где вы его **намеренно** не создали.
2. **Заметить:** рост CPU/disk в **mongostat**, хвост коллекций в **mongotop**.
3. **Найти:** включить **profiler** или смотреть **логи** slow ops; для конкретного запроса — **`explain("executionStats")`**.
4. **Исправление:** добавить индекс (пример), перепроверить план.

```javascript
use lab

// Нагрузка: вставка (пример)
for (let i = 0; i < 5000; i++) {
  db.events.insertOne({ traceId: `t-${i}`, payload: { n: i } })
}

// Плохой запрос без индекса по payload.n
db.events.find({ "payload.n": 2500 }).explain("executionStats")

// Исправление
db.events.createIndex({ "payload.n": 1 })
db.events.find({ "payload.n": 2500 }).explain("executionStats")
```

---

## Production checklist (диагностика)

| # | Практика |
|---|----------|
| 1 | **Runbook:** lag → сеть/диск/запросы; медленно → explain + индексы |
| 2 | Profiler и полный лог slow — **временно**, с лимитами |
| 3 | Не оставлять «вечный» **level 2** profiler на бою |
| 4 | После инцидента — **постмортем**: индекс, запрос, ёмкость |
| 5 | staging **должен** повторять объём/паттерн запросов хотя бы приближённо |

---

## Дополнительные материалы

- [mongostat](https://www.mongodb.com/docs/database-tools/mongostat/)
- [mongotop](https://www.mongodb.com/docs/database-tools/mongotop/)
- [Database Profiler](https://www.mongodb.com/docs/manual/tutorial/manage-the-database-profiler/)
- [db.currentOp](https://www.mongodb.com/docs/manual/reference/method/db.currentOp/)
- [Analyze Query Performance](https://www.mongodb.com/docs/manual/tutorial/analyze-query-plan/)
