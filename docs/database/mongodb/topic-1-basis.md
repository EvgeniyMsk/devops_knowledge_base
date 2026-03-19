# 1. База: NoSQL и MongoDB

---
**MongoDB** — документная **NoSQL** БД: данные хранятся как **BSON**-документы в **коллекциях**, их объединяют **базы**. Для предсказуемой производительности в production критичны **индексы** и понимание **`explain()`**. Ниже — модель данных, типы индексов, базовый **CRUD** в **mongosh** и пример запуска в **Docker**.

---

## NoSQL и зачем MongoDB

| Идея | Комментарий |
|------|----------------|
| **Гибкая схема** | Документы в одной коллекции могут иметь разные поля (с оговорками по дисциплине приложения) |
| **Документ как единица** | Удобно для JSON-подобных доменов, вложенных структур |
| **Горизонтальное масштабирование** | Шардинг и репликационные наборы — отдельные большие темы |

Когда **не** упрощать выбор: нужны сложные **мультистрочные транзакции** во всех сервисах и жёсткая **нормализация** — часто смотрят на реляционные СУБД / смешанную архитектуру.

---

## Структура: database → collection → document

```
mongodb://host:27017
 └── database (например, shop)
      └── collection (orders)
           └── document { _id: ObjectId(...), orderId: 1, items: [...] }
```

- **`_id`** — первичный ключ; при отсутствии генерируется **`ObjectId`**.
- **BSON** — бинарное представление JSON с доп. типами (`Date`, `Decimal128`, `Binary`, …).

---

## CRUD (mongosh)

Предположим, контейнер слушает **`localhost:27017`**.

```javascript
use shop

// Create
db.orders.insertOne({
  orderId: "ORD-1001",
  userId: "user-42",
  total: 99.9,
  createdAt: new Date(),
})

// Read
db.orders.findOne({ orderId: "ORD-1001" })
db.orders.find({ userId: "user-42" }).sort({ createdAt: -1 }).limit(5)

// Update
db.orders.updateOne(
  { orderId: "ORD-1001" },
  { $set: { status: "paid" }, $currentDate: { paidAt: true } },
)

// Delete
db.orders.deleteOne({ orderId: "ORD-1001" })
```

Best practice: в приложениях предпочитать **драйвер** (Node, Go, Java…) с **явными типами** и пулами соединений; `mongosh` — для операторки и отладки.

---

## Индексы

Без подходящего индекса запросы по большим коллекциям превращаются в **COLLSCAN** — плохо для latency и нагрузки на диск.

| Тип | Назначение |
|-----|------------|
| **Single field** | Равенство/сортировка по одному полю |
| **Compound** | Фильтр по нескольким полям; **порядок** ключей и направление (`1` / `-1`) важны для покрытия sort + filter |
| **TTL** | Автоудаление документов по полю-дате (например сессии, временные события) |
| **Text** | Полнотекстовый поиск по строковым полям; ограничения по числу text-index на коллекцию |

### Примеры создания

```javascript
use shop

// Одиночный
db.orders.createIndex({ orderId: 1 }, { unique: true })

// Составной (пример: пользователь + дата)
db.orders.createIndex({ userId: 1, createdAt: -1 })

// TTL: поле expireAt — время жизни документа
db.sessions.createIndex({ expireAt: 1 }, { expireAfterSeconds: 0 })

// Text (один text-индекс на коллекцию — учтите ограничения)
db.articles.createIndex({ title: "text", body: "text" })
```

Production: проектируйте индексы под **реальные** фильтры и сортировки; лишние индексы замедляют **запись** и занимают RAM.

---

## `explain()`

Оценка плана выполнения (в новых версиях чаще **`executionStats`**):

```javascript
db.orders.find({ userId: "user-42" }).sort({ createdAt: -1 }).explain("executionStats")
```

Смотрите **`winningPlan.inputStage.stage`** (например `IXSCAN` vs `COLLSCAN`) и **`executionStats.totalDocsExamined`**.

---

## Практика: MongoDB в Docker

```bash
docker run -d --name mongo-dev \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=REDACTED \
  mongo:7
```

Подключение:

```bash
mongosh "mongodb://admin:REDACTED@localhost:27017/admin"
```

В production **не** выставляйте Mongo в интернет без **TLS**, **аутентификации**, сетевых ACL и бэкапов.

---

## Production checklist (вводный)

| # | Практика |
|---|----------|
| 1 | Индексы под профиль запросов + периодический **`explain`** |
| 2 | Ограничить TTL и text-индекты осознанно |
| 3 | Мониторинг **opcounters**, **connections**, задержек репликации (при наличии) |
| 4 | Секреты не в репозитории; роли пользователей БД по **least privilege** |
| 5 | План миграций схемы документа (версионирование полей, совместимость) |

---

## Дополнительные материалы

- [MongoDB — документация](https://www.mongodb.com/docs/)
- [Indexes](https://www.mongodb.com/docs/manual/indexes/)
- [db.collection.explain()](https://www.mongodb.com/docs/manual/reference/method/cursor.explain/)

