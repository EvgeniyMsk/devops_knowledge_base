# 9. Продвинутые темы (middle+ → senior)

---
Разбор возможностей уровня **прикладной архитектуры**: **Change Streams** для событий в реальном времени, **мультидокументные транзакции**, топологии **несколько регионов**, выбор **Atlas** vs **self-hosted**, скетчи **автоматизации** (Ansible, Terraform). Акцент на ограничениях и практике **production**.

---

## Change Streams

Поток изменений на уровне **коллекции** или **БД**: приложение подписывается и получает события **`insert` / `update` / `replace` / `delete`** (и др.) из **oplog**, без внешнего poll’инга всей коллекции.

**Когда уместно:** синхронизация с поисковым индексом, кэшем, audit trail, интеграции «в реальном времени» при **идемпотентном** потребителе.

**Ограничения и best practices:**

| Аспект | Комментарий |
|--------|-------------|
| **Доступность** | На **primary**; при failover поток прерывается — драйвер переподключается, нужен **resume token** |
| **Нагрузка** | Большой поток изменений → выше нагрузка на replication; фильтруйте **`$match`** |
| **Идемпотентность** | Повтор доставки возможен; потребитель не должен ломаться от дубликатов |

### Пример (mongosh)

```javascript
// Подписка только на вставки в shop.orders (иллюстрация; в приложении — драйвер + async цикл)
const pipeline = [{ $match: { operationType: "insert" } }]
const options = { fullDocument: "updateLookup" }

const stream = db.orders.watch(pipeline, options)

// В скрипте: stream.hasNext() / stream.next() пока нужно
// Реальное приложение хранит resumeAfter / startAfter после рестарта
```

**Production:** используйте **`startAfter` / `resumeAfter`** из последнего токена; тестируйте поведение при **выборах** primary и длинном простое consumer’а.

---

## Транзакции (multi-document)

С **MongoDB 4.0+** на replica set доступны **ACID-транзакции** между несколькими документами/коллекциями в **одной сессии**. По умолчанию транзакции **короткие**; долгие удерживают ресурсы и повышают риск конфликтов (`TransientTransactionError`, `UnknownTransactionCommitResult`).

**Best practices:**

- По возможности проектируйте **атомарность на уровне документа** (`update` с вложенными массивами) — проще и быстрее.
- Транзакции — когда нужна согласованность между **несколькими** документами и это оправдано сметой latency.
- **`writeConcern: "majority"`** для коммита при необходимости устойчивости (согласуйте с продуктом).
- Обрабатывайте **retry** коммита по документации драйвера.

### Пример логики (mongosh-стиль)

```javascript
const session = db.getMongo().startSession()

session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority", j: true },
})

try {
  db.orders.insertOne({ _id: "ORD-1", status: "pending" }, { session })
  db.inventory.updateOne({ sku: "A" }, { $inc: { qty: -1 } }, { session })
  session.commitTransaction()
} catch (e) {
  session.abortTransaction()
  throw e
} finally {
  session.endSession()
}
```

На **sharded cluster** транзакции поддерживаются (см. ограничения версии); проверяйте [документацию](https://www.mongodb.com/docs/manual/core/transactions/) под вашу версию.

---

## Мультирегионные кластеры

Типичные цели: **DR** (аварийное переключение в другой регион), чтение **ближе к пользователю**, юридическое **хранение данных в регионе**.

| Идея | Комментарий |
|------|-------------|
| **Replica set в нескольких досягаемостях** | **Priority** и **votes** задают предпочтительного primary; избегайте split-brain при потере связи между сайтами |
| **Latency записи** | Запись идёт в **primary**; «удалённый» primary дороже по RTT |
| **Zones** | Правила размещения **members/shards** по регионам и зонам |
| **Read preferences** | `nearest` / региональные вторичные — со сдвигом консистентности |

**Production:** моделировать **PARTITION** сети заранее; согласовать **RPO/RTO**; для глобальных схем часто смотрят **MongoDB Atlas** (Global Writes / региональные кластеры) или тщательно спроектированный self-hosted.

---

## Atlas vs self-hosted

| Критерий | **Atlas** (managed) | **Self-hosted** |
|----------|---------------------|-----------------|
| Операции | Патчи, бэкапы, масштаб — в продукте | Всё на команде: K8s/VM, мониторинг, DR |
| Сетевая модель | VPC peering, Private Endpoint | Полный контроль, больше ответственности |
| Стоимость | Предсказуемый Opex по тарифу | CapEx + зарплата инженеров |
| Compliance | Сертификации облака | Контур заказчика |
| Гибкость | Ограничения версии/фич облака | Любые плагины/тонкая настройка при компетенции |

**Best practice:** Atlas — когда нужно **ускорить time-to-market** и снять рутину; self-hosted — жёсткие **on-prem** требования, нестандартная среда или экономия на масштабе при зрелой платформе.

---

## Автоматизация: Ansible и Terraform

### Ansible

- Модули/роли для установки пакетов, раскладки **`mongod.conf`**, **`keyFile`**, systemd, создания пользователей (**куски `mongodb_user`** в коллекции и т.д. — по коллекции `community.mongodb`).
- Идемпотентность: повторный прогон не должен ломать кластер.

```yaml
# Идея play: шаблон mongod.conf + перезапуск handler
- name: Deploy mongod config
  ansible.builtin.template:
    src: mongod.conf.j2
    dest: /etc/mongod.conf
  notify: Restart mongod
```

### Terraform

- **MongoDB Atlas Provider**: проекты, кластеры, IP access list, пользователи БД как **код**.
- Альтернатива: Terraform для **инфраструктуры** (VPC, K8s) + Helm/оператор для MongoDB внутри кластера.

```hcl
# Иллюстрация: провайдер atlas (имена ресурсов зависят от версии провайдера)
terraform {
  required_providers {
    mongodbatlas = {
      source = "mongodb/mongodbatlas"
    }
  }
}
# resource "mongodbatlas_project" "shop" { ... }
# resource "mongodbatlas_cluster" "main" { ... }
```

**Production:** секреты через **Vault** / **TF Cloud** variables; **state** с блокировкой; изменения кластера только через pipeline, не руками в обход Iac.

---

## Краткий чеклист

| # | Практика |
|---|----------|
| 1 | Change Streams — **resume token**, устойчивость к failover |
| 2 | Транзакции — **короткие**, retry, по возможности модель без них |
| 3 | Мультирегион — **выборы**, priority, зоны и сетевой **PARTITION** план |
| 4 | Atlas vs self-hosted — решение зафиксировать в **ADR** |
| 5 | Ansible/Terraform — не хранить пароли в репозитории |

---

## Дополнительные материалы

- [Change Streams](https://www.mongodb.com/docs/manual/changeStreams/)
- [Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- [Replica Sets Across Multiple Datacenters](https://www.mongodb.com/docs/manual/core/rasc/)
- [MongoDB Atlas](https://www.mongodb.com/atlas)
- [Terraform MongoDB Atlas Provider](https://registry.terraform.io/providers/mongodb/mongodbatlas/latest/docs)
