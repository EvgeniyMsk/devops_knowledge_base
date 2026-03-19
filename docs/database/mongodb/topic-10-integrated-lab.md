# 10. Интеграционная практика: production-like стенд

---
Финальный этап обучения — **собрать стенд**, близкий к production, и **прогнать сценарии** отказов. Ниже — состав стенда (**replica set**, **шардирование**, **мониторинг**, **бэкапы**, **Kubernetes**), минимальные команды и чеклист упражнений: падение ноды, деградация диска, нагрузка, восстановление.

---

## Цель стенда

| Компонент | Зачем на стенде |
|-----------|------------------|
| **Replica set (3 data-узла)** | Выборы primary, lag, переживание потери одного узла |
| **Sharded cluster (2 шарда)** | `mongos`, config servers, **balancer**, маршрутизация |
| **Prometheus + Grafana** | Метрики и алерты как в [разделе про мониторинг](topic-5-monitoring.md) |
| **Стратегия бэкапа** | Доказать **restore**, а не только «дамп создаётся» |
| **Развёртывание в Kubernetes** | StatefulSet, PVC, сервисы, сетевые имена для RS |

**Best practice:** в лаборатории допускаются упрощения (один namespace, без TLS), но **порядок операций** (инициализация RS, порядок шардов) должен повторять документацию.

---

## Вариант A: Kubernetes (идея архитектуры)

1. **Три StatefulSet** или один STS с `replicas: 3` для первого replica set (**rs0**).
2. Отдельные рабочие нагрузки для **config servers** (реплика), **два шарда** (каждый — мини-RS или один `mongod` на шард для учебы), **`mongos`** Deployment.
3. **Headless Service** для DNS `mongo-0.mongo`, `mongo-1.mongo` внутри namespace.
4. **PVC** на каждый pod с data; **StorageClass** с `WaitForFirstConsumer`, если важна топология.

```yaml
# Фрагмент: подсказка по именам для rs.initiate (не готовый кластер целиком)
# У членов replica set host в members[] должен совпадать с тем, как узлы видят друг друга
# Например: mongo-0.mongo.mongodb-lab.svc.cluster.local:27017
```

После запуска подов — **один** `mongosh` в job/initContainer выполняет `rs.initiate`, затем для шардов — `sh.addShard`, `sh.enableSharding`, `sh.shardCollection` (см. [шардирование](topic-3-sharding.md)).

**Production:** реальный кластер часто разворачивают через **оператор** или Helm-чарт с поддержанием community; ручная сборка учит только основам.

---

## Вариант B: Docker Compose (быстрее для ПК)

Несколько сервисов `mongod` с `--replSet`, отдельные порты, общий **keyFile** через volume. Шард-кластер: контейнеры **config**, **shard1**, **shard2**, **mongos** с `--configdb`. Точные `compose`-файлы зависят от версии — ориентир: [официальные примеры](https://github.com/mongo) и ваш внутренний шаблон.

---

## Мониторинг: минимум на стенде

```yaml
# prometheus.yml — тот же job, что в материале по мониторингу
scrape_configs:
  - job_name: mongodb-lab
    static_configs:
      - targets: ["mongodb-exporter.lab:9216"]
```

Импорт дашборда в Grafana; проверьте, что при записи в коллекцию растут **opcounters**.

---

## Бэкап: зафиксировать процедуру

```bash
# Логический дамп учебной БД (с хоста с mongodump)
mongodump --uri="mongodb://user:pass@mongos:27017/?authSource=admin" \
  --db=labdb --gzip --archive=/backup/labdb_$(date +%Y%m%d_%H%M).gz

# Проверка восстановления в пустую тестовую БД labdb_restore
mongorestore --uri="mongodb://..." --gzip --archive=/backup/labdb_....gz \
  --nsFrom='labdb.*' --nsTo='labdb_restore.*'
```

**Best practice:** один раз **намеренно** «сломайте» коллекцию в **labdb_restore** и восстановите из архива — это и есть ценность стенда.

---

## Сценарии для прогона

### 1. Падение ноды

- Остановите pod/process **текущего secondary** → кластер жив, **primary** прежний.
- Остановите **primary** → дождитесь **нового PRIMARY** (`rs.status()` / метрики), убедитесь, что клиент с seed-list восстановил запись.

```javascript
// После сценария
rs.status()
```

### 2. Деградация диска (эмуляция)

На лабораторном ВМ: **throttle** I/O (`iodelay` / `blkio` в cgroup / облачный volume с низким IOPS). Наблюдайте **replication lag** и **latency** в Grafana.

**Production:** такие тесты — только на **изолированном** стенде.

### 3. Нагрузка

- Простой цикл вставок / **`mongosh`** / **`mongo-benchmarking`** скрипты.
- Включите **profiler** на короткое время или смотрите **slowms** в логах.

```javascript
// Пример нагрузки (mongosh)
for (let i = 0; i < 20000; i++) {
  db.loadtest.insertOne({ i, ts: new Date() })
}
```

### 4. Восстановление

- Сценарий **«удалили БД по ошибке»**: restore из последнего **mongodump** или снапшота тома + при необходимости догон по **oplog** (если настроите PITR в лаборатории).

---

## Чеклист «стенд принят»

| # | Критерий |
|---|-----------|
| 1 | RS переживает падение **одного** узла без ручного вмешательства |
| 2 | `mongos` маршрутизирует запись; `sh.status()` показывает **два шарда** |
| 3 | В Grafana виден **lag** и **opcounters** |
| 4 | Есть **письменная** процедура backup/restore и **один** успешный тест |
| 5 | Runbook на 1 страницу: кто primary, как остановить узел, как откатить релиз |

---

## Дополнительные материалы

- [Deploy Sharded Cluster](https://www.mongodb.com/docs/manual/tutorial/deploy-shard-cluster/)
- [Kubernetes Operator](https://www.mongodb.com/docs/kubernetes-operator/stable/)
- [MongoDB в Docker](https://hub.docker.com/_/mongo)
