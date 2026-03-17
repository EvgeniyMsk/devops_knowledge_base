# 5. Мониторинг и алертинг PostgreSQL

---
Мониторинг ключевых метрик (подключения, replication lag, vacuum, диск, WAL, блокировки и deadlock) и алерты без лишнего шума. Инструменты: postgres_exporter + Prometheus, Grafana, системные представления (pg_stat_activity, pg_locks). В разделе — примеры запросов и правил алертов с комментариями и best practices для production.

---

## 12. Метрики: что мониторить

### Connections

Число подключений к экземпляру не должно приближаться к **max_connections** — иначе новые подключения получают «too many connections». Мониторить: текущее число по БД и по состоянию (active, idle, idle in transaction).

```sql
-- Текущее число подключений и лимит
SELECT count(*), (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_conn
FROM pg_stat_activity;

-- По базе и состоянию
SELECT datname, state, count(*) FROM pg_stat_activity GROUP BY datname, state ORDER BY count DESC;
```

В Prometheus **postgres_exporter** отдаёт метрику **pg_stat_activity_count** по state/datname; алерт: `pg_stat_activity_count / max_connections > 0.8`.

### Replication lag

Отставание реплики от primary по байтам WAL или по времени воспроизведения. Большой lag — риск потери данных при failover и задержка чтения на реплике.

```sql
-- На PRIMARY: lag в байтах по репликам
SELECT application_name, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

```sql
-- На STANDBY: задержка воспроизведения (секунды)
SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS replay_lag_seconds;
```

postgres_exporter: **pg_replication_lag** (в байтах или секундах в зависимости от настроек). Алерт: lag > N секунд или > M байт.

### Vacuum status

Следить, что autovacuum успевает: не накапливаются ли мёртвые строки, не отстаёт ли возраст транзакций (wraparound). Метрики: n_dead_tup, n_live_tup, last_autovacuum, возраст datfrozenxid.

```sql
-- Таблицы с большим числом мёртвых строк и давно не вакуумившиеся
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum, last_vacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC LIMIT 20;
```

```sql
-- Возраст самой старой незамороженной транзакции (предупреждение при > 150M–200M)
SELECT max(age(datfrozenxid)) AS max_frozen_age FROM pg_database;
```

postgres_exporter даёт метрики по **pg_stat_user_tables** (dead tuples, last vacuum); кастомный запрос или отдельный экспортер — для age(datfrozenxid). Алерт: dead_pct выше порога, last_autovacuum давно не обновлялся, frozen age приближается к порогу wraparound.

### Disk usage

Свободное место на разделах с данными БД и WAL. При нехватке места PostgreSQL может отказаться писать WAL или падать.

- **На сервере:** метрики с node_exporter (например, **node_filesystem_avail_bytes**) по mountpoint каталога данных и pg_wal.
- **Размер БД и таблиц** — для планирования:

```sql
SELECT pg_database.datname, pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database ORDER BY pg_database_size(pg_database.datname) DESC;
```

```sql
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;
```

Алерт: свободное место на диске < 15% (или абсолютный порог).

### WAL generation

Скорость генерации WAL (байт/сек) влияет на заполнение диска и нагрузку на репликацию/архив. Метрики: прирост позиции WAL во времени (через postgres_exporter или кастомный скрипт). Резкий рост — признак тяжёлой записи или долгих транзакций.

```sql
-- Текущая позиция WAL (для расчёта прироста между замерами)
SELECT pg_current_wal_lsn();
```

### Locks и deadlocks

Долгие блокировки блокируют другие запросы; deadlock завершается прерыванием одной из транзакций. Мониторить: число ожидающих блокировку, длительность ожидания, факты deadlock (счётчик в pg_stat_database).

```sql
-- Активные блокировки и кто ждёт
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query,
       blocking_activity.query AS blocking_query,
       now() - blocked_activity.query_start AS blocked_duration
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

```sql
-- Число deadlock'ов по БД (накопленный счётчик)
SELECT datname, deadlocks FROM pg_stat_database;
```

postgres_exporter: **pg_stat_database_deadlocks**; для блокировок — кастомный запрос или алерт по длительности запросов в pg_stat_activity (state = 'active' и долгое время). Алерт: deadlocks > 0 за период; долго висящие blocking-запросы.

### Idle in transaction

Сессии в состоянии **idle in transaction** держат соединение и блокировки (в т.ч. невидимые для VACUUM строки), не делая полезной работы. Долгие idle in transaction — частая причина bloat и блокировок.

```sql
-- Сессии idle in transaction дольше 5 минут
SELECT pid, usename, application_name, state, now() - state_change AS idle_duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < now() - interval '5 minutes';
```

Алерт: число таких сессий выше порога или максимальная длительность idle in transaction > N минут.

---

## Инструменты: postgres_exporter, Prometheus, Grafana

**postgres_exporter** — экспортер метрик PostgreSQL для Prometheus. Запускается рядом с БД или в сети с доступом к ней; подключается к одной или нескольким БД по DSN, отдаёт метрики по активности, репликации, размерам, блокам и т.д. Рекомендуется отдельный пользователь с ограниченными правами (только чтение системных каталогов и представлений).

```yaml
# Пример флага для подключения (переменные окружения или аргументы)
# DATA_SOURCE_NAME="postgresql://monitor:password@localhost:5432/postgres?sslmode=disable"
# Запуск: ./postgres_exporter
```

В Prometheus добавляют scrape job на порт postgres_exporter (обычно 9187). В Grafana используют готовые дашборды (например, «PostgreSQL» по ID 9628) или строят свои на основе метрик postgres_exporter и node_exporter.

**pg_stat_activity и pg_locks** — при необходимости детальной диагностики запросы выполняют вручную или через скрипты/кастомные метрики (например, текст запроса блокирующей сессии, длительность).

---

## 13. Алерты: реальные и без шума

Алерты должны срабатывать по действительным проблемам и вести к действию (runbook). Избегать порогов, при которых алерт «дёргается» при нормальной нагрузке (например, кратковременный всплеск соединений). Использовать **for** в Prometheus, чтобы не алертить на короткий скачок.

### Примеры правил алертинга (Prometheus)

Ниже — идеи выражений; точные имена метрик зависят от установки postgres_exporter и версии.

```yaml
# Replication lag > 60 секунд (на standby или с метрики primary)
# Пример: pg_replication_lag > 60
- alert: PostgresReplicationLagHigh
  expr: pg_replication_lag > 60
  for: 5m
  labels: { severity: warning }
  annotations:
    summary: "PostgreSQL replication lag high on {{ $labels.instance }}"
```

```yaml
# Autovacuum не успевает: много мёртвых строк и last_autovacuum старый
# Упрощённо: таблицы с n_dead_tup выше порога (если экспортер отдаёт такие метрики)
# Или кастомная метрика по запросу к pg_stat_user_tables
- alert: PostgresDeadTuplesHigh
  expr: pg_stat_user_tables_n_dead_tup > 100000
  for: 30m
  labels: { severity: warning }
  annotations:
    summary: "PostgreSQL table {{ $labels.relname }} has many dead tuples"
```

```yaml
# Свободное место на диске < 15% (node_exporter, раздел с данными)
- alert: DiskSpaceLow
  expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 15
  for: 5m
  labels: { severity: critical }
  annotations:
    summary: "Disk space below 15% on {{ $labels.instance }} {{ $labels.mountpoint }}"
```

```yaml
# Слишком много idle in transaction (кастомная метрика или текст-запрос)
# Пример: count по pg_stat_activity где state = 'idle in transaction' и state_change старше 5 min
- alert: PostgresIdleInTransactionTooLong
  expr: pg_stat_activity_idle_in_transaction_count > 5
  for: 10m
  labels: { severity: warning }
  annotations:
    summary: "PostgreSQL: many long idle in transaction sessions"
```

Дополнительно: алерт на **connections / max_connections > 0.8**; на **deadlocks** за последние N минут; на **replication slot inactive** (реплика отвалилась, слот не освобождён). Для каждой алерты описать в аннотациях или в runbook, что делать (проверить запросы, убить долгие idle in transaction, увеличить место на диске, проверить реплику и т.д.).

---

## Best practices для production

| Область | Рекомендация |
|--------|---------------|
| **Метрики** | Обязательно: connections, replication lag, vacuum (dead tuples / last autovacuum), disk space, при необходимости WAL rate и locks. |
| **Экспортер** | Отдельная роль с минимальными правами; не под суперпользователем. |
| **Алерты** | Использовать for (5–10 мин), чтобы не шуметь на кратковременных всплесках; severity разделять (critical — реагировать сразу, warning — в рабочее время). |
| **Runbook** | У каждой алерты — краткое описание действий (проверить pg_stat_activity, реплику, vacuum, диск). |
| **Idle in transaction** | Мониторить и алертить; в приложении закрывать транзакции или ограничивать таймаут idle in transaction (idle_in_transaction_session_timeout). |

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Один дашборд по кластеру/инстансу** | Connections, lag, vacuum, disk, топ запросов — в одном месте. |
| **Алерты с for и осмысленными порогами** | Меньше ложных срабатываний, понятная реакция. |
| **idle_in_transaction_session_timeout** | Автоматическое обрывание долгих idle in transaction (PostgreSQL 9.6+). |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Алертить на каждую метрику без for** | Шум при нормальных колебаниях. | Добавить for и поднять пороги. |
| **Нет алерта на диск** | Заполнение диска — падение или остановка записи. | Алерт на свободное место (например, < 15%). |
| **Игнорировать idle in transaction** | Bloat и блокировки. | Мониторить, алертить, настроить timeout в приложении/БД. |

---

## Дополнительные материалы

- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter)
- [PostgreSQL — Monitoring Stats](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [pg_stat_activity](https://www.postgresql.org/docs/current/monitoring-stats.html#PG-STAT-ACTIVITY-VIEW)
- [pg_locks](https://www.postgresql.org/docs/current/view-pg-locks.html)
- [2. Эксплуатация и стабильность](topic-2-operations-stability.md) — VACUUM
- [3. High Availability и масштабирование](topic-3-ha-scaling.md) — репликация, lag
