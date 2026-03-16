# 1. База PostgreSQL: архитектура, конфигурация и безопасность

**Цель:** уверенно понимать, как устроен PostgreSQL: процессная модель, shared buffers, WAL, MVCC, ключевые параметры конфигурации (postgresql.conf, pg_hba.conf), роли и привилегии. В разделе — примеры кода и SQL с комментариями и best practices для production.

---

## 1. Архитектура PostgreSQL

### Процессная модель

PostgreSQL работает как **мультипроцессное** приложение (не потоки в одном процессе):

- **postmaster** — главный процесс; слушает порт, принимает подключения, порождает **backend-процессы** по одному на каждое соединение.
- **Backend processes** — обрабатывают запросы клиента; каждый подключённый клиент = один backend.
- Вспомогательные процессы: **checkpointer**, **background writer (bgwriter)**, **walwriter**, **autovacuum launcher**, **autovacuum workers**, **stats collector** и др.

```bash
# Посмотреть процессы PostgreSQL (на сервере БД)
ps aux | grep postgres
# Типичный вывод: postgres (postmaster), postgres: checkpointer, postgres: bgwriter,
# postgres: walwriter, postgres: autovacuum launcher, postgres: ... worker, postgres: client backend
```

### shared_buffers, WAL, bgwriter, checkpointer

- **shared_buffers** — общий буфер в RAM для страниц данных; сюда читаются и сюда пишутся страницы с диска. Размер задаётся в `postgresql.conf`; типично 25% RAM сервера (но не единственный критерий).
- **WAL (Write-Ahead Log)** — журнал предзаписи; изменения сначала пишутся в WAL, затем в страницы данных. Обеспечивает долговечность и возможность восстановления после сбоя. **walwriter** сбрасывает WAL на диск; **checkpointer** периодически помечает «до этой точки WAL данные сброшены в данные» и триггерит запись грязных страниц из shared_buffers.
- **bgwriter** — фоново записывает грязные страницы из shared_buffers на диск, чтобы под нагрузкой не возникало пиков I/O при checkpoint.

```sql
-- Текущая активность и фоновые процессы (подключиться к БД и выполнить)
SELECT pid, usename, application_name, state, query
FROM pg_stat_activity
WHERE backend_type = 'client backend' OR backend_type = 'background';
```

```sql
-- Статистика background writer и checkpoint (полезно для тюнинга)
SELECT * FROM pg_stat_bgwriter;
-- checkpoints_timed / checkpoints_req — сколько checkpoint'ов по расписанию и по требованию
-- buffers_checkpoint, buffers_backend, buffers_alloc — откуда записывались буферы
```

### MVCC (Multi-Version Concurrency Control)

**MVCC** — ключевая тема: каждая строка может иметь несколько версий (версии видны разным транзакциям в зависимости от изоляции и момента снимка). Нет блокировок на чтение при записи; читатели видят консистентный снимок. Версии строк хранятся в самой таблице (и в индексах); старые версии помечаются и удаляются **VACUUM** (в т.ч. autovacuum). Поля **xmin**, **xmax** в строке (и видимость по xid) определяют, видна ли версия данной транзакции.

- **VACUUM** — помечает мёртвые версии как пригодные для переиспользования; **VACUUM FULL** перезаписывает таблицу (блокирует её).
- **autovacuum** — фоновые воркеры запускают VACUUM и ANALYZE по таблицам при необходимости.

### TOAST, FSM, visibility map

- **TOAST** — большие значения (например, текст > 2 KB) хранятся в отдельных TOAST-таблицах; в основной строке остаётся указатель. Снижает размер основной таблицы и ускоряет сканы.
- **FSM (Free Space Map)** — карта свободного места в страницах таблицы; используется при вставке, чтобы быстрее найти подходящую страницу.
- **Visibility map** — битовая карта страниц, где все строки видны всем активным транзакциям; VACUUM может пропускать такие страницы при сканировании (index-only scan использует VM для нечтения таблицы, если все видимые строки в индексе).

Практика: после изучения теории посмотреть `pg_stat_user_tables` (n_dead_tup, n_live_tup, last_vacuum, last_autovacuum) и при необходимости подстроить autovacuum.

---

## 2. Конфигурация и параметры

### Файлы конфигурации

| Файл | Назначение |
|------|------------|
| **postgresql.conf** | Основные параметры: память, WAL, checkpoint, autovacuum, логирование и т.д. |
| **pg_hba.conf** | Правила доступа: с каких хостов и под каким пользователем разрешено подключаться (host, user, method: trust, scram-sha-256, md5 и т.д.). |
| **pg_ident.conf** | Маппинг системных пользователей ОС на роли PostgreSQL (редко меняется). |

Изменения в **postgresql.conf** вступают в силу после **restart** (некоторые параметры) или **reload** (SIGHUP). Изменения в **pg_hba.conf** — после **reload** (новые подключения используют новые правила).

```bash
# Перезагрузить конфигурацию без рестарта (применить изменения, поддерживающие reload)
pg_ctl reload -D /var/lib/postgresql/data
# или из SQL (суперпользователь):
# SELECT pg_reload_conf();
```

### Ключевые параметры (postgresql.conf)

```ini
# Память
shared_buffers = 256MB          # Общий буфер; при старте выделяется сразу. Изменение — только рестарт.
work_mem = 4MB                 # Память на операции сортировки/хеш в одном запросе. Reload.
maintenance_work_mem = 128MB   # Память на VACUUM, CREATE INDEX и т.д. Reload.
effective_cache_size = 1GB     # Оценка для планировщика (сколько RAM доступно для кэша). Reload.

# WAL и checkpoint
wal_level = replica            # replica — для репликации; minimal — только для восстановления после сбоя. Рестарт.
max_wal_size = 1GB            # Целевой размер WAL до принудительного checkpoint. Reload.
checkpoint_timeout = 15min     # Максимальный интервал между checkpoint'ами. Reload.

# Autovacuum (в production обычно не отключать)
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min
autovacuum_vacuum_scale_factor = 0.1   # Запускать vacuum при 10% мёртвых строк
autovacuum_analyze_scale_factor = 0.05
```

**Что требует рестарта:** `shared_buffers`, `wal_level`, ряд других. Остальное часто — **reload**. Проверка: в документации или `pg_settings` (поле `context`: `postmaster` = рестарт, `sighup` = reload).

```sql
-- Параметры и контекст изменения (рестарт vs reload)
SELECT name, setting, unit, context FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'wal_level', 'max_wal_size')
ORDER BY name;
```

### Практика: менять параметры и видеть эффект

- Увеличить `work_mem` для тяжёлого запроса с сортировкой — выполнить до и после reload, сравнить план (EXPLAIN ANALYZE).
- Посмотреть `pg_stat_bgwriter`: если `checkpoints_req` часто растёт — увеличить `max_wal_size` или `checkpoint_timeout`, чтобы реже принудительный checkpoint.

---

## 3. Роли, доступы, безопасность

### Роли vs пользователи

В PostgreSQL **роль** — единая сущность: может быть «пользователем» (LOGIN), «группой» (роли, в которые входят другие роли), владельцем объектов. Отдельного понятия «пользователь» нет — это роль с правом LOGIN.

```sql
-- Создать роль с правом входа (пользователь)
CREATE ROLE app_user WITH LOGIN PASSWORD 'secure_password';

-- Роль без входа (группа для привилегий)
CREATE ROLE read_only_group NOLOGIN;

-- Выдать членство: app_user входит в read_only_group
GRANT read_only_group TO app_user;
```

### GRANT, REVOKE

Привилегии на объекты (таблицы, схемы, базы) выдаются через **GRANT** и снимаются через **REVOKE**. Минимальные привилегии — только то, что нужно приложению.

```sql
-- Разрешить выборку по таблице
GRANT SELECT ON TABLE public.orders TO app_user;

-- Разрешить выборку по всей схеме (удобно для read-only сервиса)
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_user;

-- По умолчанию новые таблицы в схеме не наследуют права — включить
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO app_user;

-- Только определённые столбцы (если не хотите открывать всю таблицу)
GRANT SELECT (id, status) ON TABLE public.orders TO app_user;

-- Отозвать привилегию
REVOKE INSERT ON TABLE public.orders FROM app_user;
```

### pg_hba.conf

Правила **pg_hba.conf** задают: с какого адреса (host), к какой базе (database), под какой ролью (user) и каким методом аутентификации (method) разрешено подключаться. Порядок правил важен — первое совпадение применяется.

```ini
# TYPE  DATABASE        USER            ADDRESS         METHOD
# Локальные подключения (сокет)
local   all             all                             trust

# Сеть: приложение с конкретного IP — SCRAM-SHA-256
host    mydb            app_user        10.0.0.0/8      scram-sha-256

# Репликация (для standby)
host    replication     replicator       10.0.0.0/8      scram-sha-256
```

После изменения — `pg_ctl reload` или `SELECT pg_reload_conf();`.

### SSL/TLS, SCRAM vs MD5

- **SSL/TLS** — шифрование соединения; настраивается параметрами `ssl = on`, сертификаты на сервере. В **pg_hba.conf** можно требовать SSL: `hostssl` вместо `host`.
- **SCRAM-SHA-256** — рекомендуемый метод хранения и проверки паролей (устойчив к перебору). **MD5** устарел; не использовать для новых установок.

```sql
-- Задать пароль (хранится в виде SCRAM при default password_encryption)
ALTER ROLE app_user WITH PASSWORD 'new_password';

-- Проверить метод шифрования паролей
SHOW password_encryption;  -- должно быть scram-sha-256
```

### DevOps-фокус: минимальные привилегии, сервисные аккаунты, ротация

- **Минимальные привилегии:** одной роли приложения — только SELECT/INSERT/UPDATE на нужные таблицы, без суперпользователя и без прав на DDL. Отдельная роль для миграций (с правом на CREATE TABLE и т.д.) с ограниченным использованием.
- **Сервисные аккаунты:** отдельная роль на сервис (например, `svc_orders_api`); пароль или сертификат хранятся в секретах (Vault, Kubernetes Secret), не в коде.
- **Ротация паролей:** смена пароля через `ALTER ROLE ... PASSWORD '...'`; обновить секрет в хранилище и перезапустить приложение. Для нулевого даунтайма — два пользователя с одинаковыми правами, переключение по очереди.

```sql
-- Пример: роль только для чтения отчётности
CREATE ROLE reporting_readonly WITH LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE mydb TO reporting_readonly;
GRANT USAGE ON SCHEMA public TO reporting_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO reporting_readonly;
```

---

## Best practices для production

| Область | Рекомендация |
|--------|--------------|
| **Память** | shared_buffers — порядка 25% RAM (но не гигантские значения без тестов); work_mem не ставить слишком большим на всю систему (умножить на max_connections). |
| **WAL** | wal_level = replica при репликации; адекватный max_wal_size, чтобы не было частых checkpoints_req. |
| **Autovacuum** | Не отключать; при больших таблицах подстроить scale_factor и naptime по pg_stat_user_tables. |
| **Безопасность** | pg_hba — минимальный набор хостов и пользователей; везде scram-sha-256; SSL для сетевых подключений. |
| **Роли** | Отдельные роли на приложение/сервис; только нужные GRANT; пароли в секретах, ротация по процессу. |

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Отдельная роль на сервис** | Минимальные права, аудит по роли. |
| **SCRAM + SSL для сетевого доступа** | Защита пароля и трафика. |
| **Мониторинг pg_stat_* и autovacuum** | Своевременное выявление долгих запросов и накопления мёртвых строк. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Один суперпользователь для приложений** | Любая утечка — полный доступ. | Отдельные роли с GRANT только на нужные объекты. |
| **trust в pg_hba для сети** | Доступ без пароля с любых хостов. | host/hostssl + scram-sha-256, ограничить ADDRESS. |
| **Отключение autovacuum «чтобы не мешал»** | Рост мёртвых строк, раздувание таблиц и индексов. | Настроить autovacuum, не отключать. |

---

## Дополнительные материалы

- [PostgreSQL — Server Configuration](https://www.postgresql.org/docs/current/runtime-config.html)
- [PostgreSQL — Client Authentication (pg_hba.conf)](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)
- [PostgreSQL — Roles and Privileges](https://www.postgresql.org/docs/current/user-manag.html)
- [PostgreSQL — MVCC](https://www.postgresql.org/docs/current/mvcc.html)
