# 3. High Availability и масштабирование: репликация, failover, connection pooling

---
Streaming replication (синхронная и асинхронная), replication slots, lag; инструменты failover (Patroni, repmgr, облачные решения); connection pooling (PgBouncer, Pgpool) и устранение «too many connections». В разделе — примеры кода и конфигурации с комментариями и best practices для production.

---

## 7. Репликация

### Streaming replication

**Streaming replication** — реплика standby непрерывно получает WAL по TCP от primary и применяет его. На primary включают **wal_level = replica** (или higher); на standby в каталоге данных задают **primary_conninfo** и при необходимости **restore_command** для архива WAL. Standby может быть только для чтения (read replica) или кандидатом на promote при failover.

### Синхронная и асинхронная репликация

- **Асинхронная (по умолчанию):** primary отвечает клиенту после записи WAL у себя; реплика может отставать. При падении primary возможна потеря последних транзакций.
- **Синхронная:** primary ждёт подтверждения от одной или нескольких реплик (до записи WAL на standby), затем отвечает клиенту. Гарантия: коммит не потеряется при падении primary, если реплика успела применить WAL. Минус — задержка и зависимость от доступности реплики.

```ini
# postgresql.conf на PRIMARY
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
# Синхронная репликация: ждать подтверждения от standby из списка
synchronous_commit = on
synchronous_standby_names = 'standby1'   # или 'FIRST 1 (standby1, standby2)'
```

```ini
# postgresql.auto.conf или recovery-файлы на STANDBY (PostgreSQL 12+)
primary_conninfo = 'host=primary.example.com port=5432 user=replicator password=... application_name=standby1'
```

### Replication slots

**Replication slot** — именованный слот на primary, который хранит позицию WAL до которой standby получил данные. Primary не удаляет сегменты WAL, нужные слотам. Защищает от потери WAL при длительном отставании standby, но при «мёртвом» standby слот может задерживать удаление WAL и заполнить диск.

```sql
-- На PRIMARY: создать слот для standby
SELECT * FROM pg_create_physical_replication_slot('standby1_slot');

-- Список слотов и задержка
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;
```

На standby в **primary_conninfo** указывают **primary_slot_name = 'standby1_slot'**. В production мониторить активные слоты и отстающие; при отказе standby удалять слот или переключать на другую реплику.

### Lag и его причины

**Replication lag** — отставание реплики от primary (по байтам WAL или по времени). Причины: медленный диск на standby, нехватка CPU на apply, одна долгая транзакция на primary (WAL не переключается), сетевая задержка, конфликт репликации (например, lock на standby). Мониторинг — на primary и на standby.

```sql
-- На PRIMARY: отставание реплик (байты и время)
SELECT
  client_addr,
  application_name,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

```sql
-- На STANDBY: задержка воспроизведения
SELECT now() - pg_last_xact_replay_timestamp() AS replay_lag;
```

### Практика: поднять primary + replica

1. **Primary:** wal_level = replica, max_wal_senders, pg_hba — разрешить подключение пользователя репликации с хоста standby.
2. **Standby:** pg_basebackup с primary или копия данных; в каталоге создать **standby.signal**; в postgresql.auto.conf или в recovery-настройках задать primary_conninfo (и при необходимости primary_slot_name). Запустить — реплика подхватит поток WAL.
3. Проверить: на primary в pg_stat_replication видна реплика, lag уменьшается; на standby запросы только на чтение.

### Практика: сломать primary и проверить поведение

При «падении» primary асинхронная реплика может иметь последние транзакции не применёнными — возможна потеря данных. Синхронная реплика до падения primary уже имеет их. **Promote** standby в новый primary: на standby выполнить **pg_ctl promote** или создать триггер-файл **promote.signal**. После этого standby становится primary и принимает запись; старый primary при возврате нужно перевести в standby (пересобрать конфиг и перезапустить) или заменить. Без автоматизации (Patroni, repmgr) переключение делают вручную по runbook.

---

## 8. Failover и HA

### Инструменты

- **Patroni** — шаблон HA для PostgreSQL: etcd/Consul/ZooKeeper для распределённого состояния, автоматический выбор leader, переключение при падении primary, replication slots и конфигурация через REST API. Широко используется в self-hosted.
- **repmgr** — управление кластером реплик: регистрация узлов, failover, переключение с помощью repmgr standby promote. Проще Patroni по стеку, но без встроенного распределённого хранилища (можно комбинировать с внешним consensus).
- **Cloud-native (RDS, Cloud SQL, Azure Database):** провайдер управляет primary и репликами, автоматический или ручной failover, мониторинг lag. Концептуально те же понятия: primary, read replica, синхронный/асинхронный, точка восстановления.

### DevOps-фокус: split-brain, fencing, TTL, leader election

- **Split-brain:** после failover два узла считают себя primary и принимают запись — данные расходятся. Избегают: только один «источник истины» (например, через распределённый lock в etcd у Patroni); **fencing** — отключение/блокировка старого primary при переключении (STONITH, отказ в сети и т.д.).
- **Fencing:** гарантированное отключение старого primary от клиентов и от реплик (отзыв VIP, отключение сети, выключение инстанса в облаке), чтобы он не писал после того, как новый primary уже выбран.
- **TTL, leader election:** в Patroni лидер держит lease в etcd/Consul с TTL; при падении primary lease истекает, другой узел проводит election и становится primary. Таймауты и TTL настраивают под сеть и время обнаружения сбоя.

Best practice: документировать сценарии failover и fencing; проводить учения (chaos) в staging.

---

## 9. Connection pooling

### Проблема: Postgres плохо масштабируется по коннектам

Каждое подключение — отдельный backend-процесс с выделением памяти (work_mem и др.). При тысячах соединений нагрузка на память и контекст-переключения растёт; типичный лимит **max_connections** — сотни. Приложение с пулом из сотен соединений на инстанс создаёт «too many connections», если инстансов много или лимит занижен.

### Решения: PgBouncer (session / transaction)

**PgBouncer** — лёгкий пулер соединений: клиенты подключаются к PgBouncer, он держит ограниченный пул реальных соединений к PostgreSQL. Режимы:

- **Session** — клиент занимает одно соединение с БД на время сессии (до отключения).
- **Transaction** — соединение с БД выдаётся только на время транзакции; после COMMIT/ROLLBACK соединение возвращается в пул. Подходит для коротких транзакций и максимальной экономии соединений.
- **Statement** — соединение на один запрос (ограничения: не все режимы совместимы с prepared statements и т.д.).

```ini
# pgbouncer.ini (фрагмент)
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
reserve_pool_size = 10
```

Приложение подключается к **6432** (PgBouncer); PgBouncer держит до **default_pool_size** соединений к PostgreSQL. При **transaction** режиме 1000 клиентов укладываются в 50–60 соединений к БД.

### Pgpool (на уровне понимания)

**Pgpool-II** — пулинг + дополнительные функции: load balancing чтения по репликам, watch dog для HA. Сложнее в настройке и эксплуатации; выбирают при необходимости балансировки read-запросов по нескольким standby. Для чистой экономии соединений чаще достаточно PgBouncer.

### Практика: сравнить с/без PgBouncer

- Без пулера: поднять max_connections в Postgres, нагрузить приложение или бенчмарк (pgbench) до лимита — наблюдать «too many connections» при превышении.
- С PgBouncer: направить клиентов на PgBouncer, pool_mode = transaction, default_pool_size меньше max_connections Postgres — та же нагрузка укладывается в пул; мониторить **SHOW POOLS** в PgBouncer.

### Ошибка «too many connections»

Причина: число подключений к PostgreSQL достигло **max_connections**. Решения:

1. **Connection pooling** — приложение подключается к PgBouncer, пул соединений к Postgres ограничен (например, 50–100).
2. Увеличить **max_connections** на primary (с учётом памяти: каждое соединение ~ work_mem и др.) — временная мера; лучше ограничить пул на стороне приложения и пулера.
3. Проверить утечки соединений в приложении (незакрытые соединения, долгие транзакции); мониторить **pg_stat_activity** и длительность idle-сессий.

```sql
-- Текущие подключения по БД и состоянию
SELECT datname, state, count(*) FROM pg_stat_activity GROUP BY datname, state ORDER BY count DESC;
```

---

## Примеры конфигурации и команд

### Primary: pg_hba для репликации

```ini
# pg_hba.conf на PRIMARY
# TYPE  DATABASE        USER            ADDRESS         METHOD
host    replication     replicator      10.0.0.0/8      scram-sha-256
host    all             replicator      10.0.0.0/8      scram-sha-256
```

### Standby: standby.signal и primary_conninfo (PostgreSQL 12+)

```bash
# В каталоге данных standby создаётся пустой файл — сервер работает в режиме standby
touch $PGDATA/standby.signal
```

```ini
# postgresql.auto.conf или включить в конфиг
primary_conninfo = 'host=primary.example.com port=5432 user=replicator password=xxx application_name=standby1'
primary_slot_name = 'standby1_slot'
```

### PgBouncer: userlist.txt (auth_file)

```ini
# Формат: "username" "password" (пароль в SCRAM или md5 по auth_type)
"replicator" "xxx"
"app_user" "yyy"
```

---

## Best practices для production

| Область | Рекомендация |
|--------|--------------|
| **Репликация** | Синхронная — при требовании нулевой потери данных (1 реплика в synchronous_standby_names); мониторить lag и replication slots. |
| **Replication slots** | Не оставлять неактивные слоты надолго — риск заполнения диска WAL; при отказе standby удалять слот или переключать. |
| **Failover** | Использовать Patroni/repmgr или облачный managed service; документировать fencing и порядок переключения. |
| **Connection pooling** | PgBouncer в transaction mode перед приложением; default_pool_size меньше max_connections Postgres; мониторить SHOW POOLS. |
| **max_connections** | Не поднимать без необходимости; ограничивать через пулер и пул в приложении. |

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Пулер перед каждым приложением** | Один пул соединений к Postgres на сервис; предсказуемая нагрузка. |
| **Мониторинг lag и слотов** | Алерты на отставание реплики и на неактивные слоты. |
| **Синхронная реплика для критичных данных** | Гарантия отсутствия потери коммита при падении primary (в пределах реплики). |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Тысячи соединений напрямую к Postgres** | Исчерпание памяти и «too many connections». | PgBouncer (или аналог), ограничить пул в приложении. |
| **Забыть про replication slots** | При долгом отставании standby WAL может удаляться на primary — разрыв репликации. | Использовать слоты и мониторить их; при отказе standby удалять слот. |
| **Failover без fencing** | Старый primary продолжает принимать запись — split-brain. | Обеспечить отключение старого primary (Patroni fencing, облако). |

---

## Дополнительные материалы

- [PostgreSQL — Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [PostgreSQL — Replication Slots](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html#LOGICALDECODING-REPLICATION-SLOTS)
- [Patroni](https://patroni.readthedocs.io/)
- [repmgr](https://repmgr.org/)
- [PgBouncer](https://www.pgbouncer.org/)
- [2. Эксплуатация и стабильность](topic-2-operations-stability.md) — WAL, бэкапы
