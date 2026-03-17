# 2. Эксплуатация и стабильность: VACUUM, WAL, Backup & Restore

---
VACUUM и Autovacuum (мёртвые строки, bloat, wraparound), WAL и checkpoints (fsync, влияние на I/O), резервное копирование и восстановление (pg_dump, pg_basebackup, PITR, инструменты pgBackRest и wal-g). В разделе — примеры кода и SQL с комментариями и best practices для production.

---

## 4. VACUUM и Autovacuum

### Почему без VACUUM «всё умирает»

При **MVCC** обновления и удаления не затирают старые версии строк сразу — они остаются в таблице как **dead tuples** (мёртвые кортежи). Пока их не обработает **VACUUM**, они занимают место, раздувают индексы и замедляют сканы. Без VACUUM таблицы и индексы растут (bloat), запросы тормозят; при исчерпании **transaction ID** возможен **wraparound** — база переходит в режим только vacuum до завершения анти-wraparound vacuum.

### Dead tuples и мониторинг

```sql
-- Количество живых и мёртвых строк по таблицам (ключевая метрика для autovacuum)
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

```sql
-- Расширенная статистика по всем таблицам (включая системные)
SELECT relname, n_tup_ins, n_tup_upd, n_tup_del, n_live_tup, n_dead_tup
FROM pg_stat_all_tables
WHERE schemaname = 'public'
ORDER BY n_dead_tup DESC;
```

### Wraparound

Transaction ID в PostgreSQL ограничен 32 битами; после ~2 млрд транзакций счётчик «заворачивается». Чтобы старые строки не казались «будущими», нужен **anti-wraparound VACUUM** — он помечает старые версии как замороженные (frozen). Обычно за это отвечает autovacuum; при приближении к порогу wraparound autovacuum агрессивно вакуумит таблицы. Мониторить возраст самой старой транзакции:

```sql
-- Возраст самой старой незамороженной транзакции (предупреждение при приближении к 200M)
SELECT datname, age(datfrozenxid) AS frozen_age
FROM pg_database
WHERE datname = current_database();
```

### Параметры Autovacuum

```ini
# postgresql.conf — типичные настройки для production
autovacuum = on
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
autovacuum_vacuum_scale_factor = 0.1   # Запуск vacuum при 10% мёртвых строк
autovacuum_analyze_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 2000
```

Порог запуска vacuum: `vacuum_threshold + scale_factor * n_live_tup`. Для больших таблиц scale_factor 0.1 может давать слишком большой порог — тогда уменьшают scale_factor или задают **per-table** настройки:

```sql
-- Для очень большой таблицы: снизить порог (vacuum чаще)
ALTER TABLE big_events SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01
);
```

### Симуляция bloat и ручной VACUUM

```sql
-- Создать таблицу и нагенерировать мёртвые строки (UPDATE без изменения данных тоже создаёт версии)
CREATE TABLE t_bloat_demo (id int PRIMARY KEY, val text);
INSERT INTO t_bloat_demo SELECT i, 'x' FROM generate_series(1, 10000) i;

-- Много обновлений — много dead tuples
UPDATE t_bloat_demo SET val = val WHERE id % 2 = 0;
UPDATE t_bloat_demo SET val = val WHERE id % 2 = 0;

-- Проверить n_dead_tup
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 't_bloat_demo';

-- Ручной VACUUM (не блокирует чтение/запись, в отличие от VACUUM FULL)
VACUUM (ANALYZE) t_bloat_demo;

-- VACUUM FULL перезаписывает таблицу, требует эксклюзивной блокировки — только при сильном bloat и в окно обслуживания
-- VACUUM FULL t_bloat_demo;
```

### Best practices: VACUUM и Autovacuum

- **Не отключать autovacuum** в production; при необходимости тюнить пороги и cost, а не отключать.
- **Мониторить** `n_dead_tup`, `last_autovacuum`, возраст `datfrozenxid`; алертить на высокий dead_pct и приближение к wraparound.
- **VACUUM FULL** — только при доказанном сильном bloat и в окно обслуживания; в остальных случаях достаточно обычного VACUUM и при необходимости реиндексации.

---

## 5. WAL и Checkpoints

### Зачем нужен WAL

**WAL** обеспечивает долговечность: изменения сначала записываются в WAL и сбрасываются на диск (fsync); только затем страницы данных могут быть записаны позже. При сбое восстановление идёт по WAL от последнего checkpoint. Рост WAL ограничивается checkpoint'ами: при checkpoint грязные буферы сбрасываются на диск, и старые сегменты WAL можно переиспользовать или удалить (при архивировании — копировать в архив).

### fsync и synchronous_commit

- **fsync** — сбрасывать ли изменения файлов на диск (отключение опасно при сбое питания).
- **synchronous_commit = on** — ждать подтверждения записи WAL на диск перед ответом клиенту. **synchronous_commit = off** или **local** — быстрее, но при сбое возможна потеря последних транзакций. В production обычно оставляют **on**; для некритичных нагрузок допустимо **off** с пониманием риска.

```sql
SHOW synchronous_commit;
-- Для репликации: synchronous_commit = 'on' и в primary настроен synchronous_standby_names при необходимости
```

### Влияние checkpoints на I/O

При **checkpoint** все грязные буферы до текущей точки WAL записываются на диск — возможен всплеск I/O. Параметры **max_wal_size** и **checkpoint_timeout** задают, как часто это происходит: больший max_wal_size реже принудительный checkpoint, но больше WAL на диске.

```sql
-- Статистика checkpoint'ов (частые checkpoints_req — признак нехватки WAL или нагрузки)
SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint, buffers_backend
FROM pg_stat_bgwriter;
```

### Мониторинг роста WAL

```sql
-- Текущий сегмент WAL и позиция
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());

-- Размер директории WAL (на сервере; путь из data_directory + pg_wal)
-- ls -la $PGDATA/pg_wal
```

```bash
# На сервере: размер каталога WAL
du -sh /var/lib/postgresql/data/pg_wal
```

### Ситуация «диск внезапно закончился»

Частые причины: взрывной рост WAL (долгие транзакции, репликация не успевает забирать WAL, архив не успевает), накопление сегментов при сбое архивирования. Действия:

1. Освободить место (удалить старые логи, временные файлы) или расширить диск.
2. Проверить реплику и архив: не застряла ли доставка WAL; при необходимости временно увеличить **wal_keep_size** или исправить архив.
3. Убедиться, что checkpoint может завершиться: при нехватке места для WAL сервер может не давать писать новые сегменты.

В production — мониторинг свободного места на диске с данными и WAL, алерты до заполнения.

---

## 6. Backup & Restore

### pg_dump и pg_dumpall (логический бэкап)

**Логический бэкап** — дамп SQL или custom-формат; восстанавливается через pg_restore или psql. Удобен для переноса между версиями и выборочного восстановления объектов.

```bash
# Дамп одной БД в custom-формат (сжатие, параллельный restore)
pg_dump -Fc -f /backup/mydb_$(date +%Y%m%d).dump mydb

# Дамп одной БД в SQL (для мелких БД или переноса)
pg_dump -f /backup/mydb.sql mydb

# Дамп всех БД + глобальные объекты (роли, табличные пространства)
pg_dumpall -f /backup/all_$(date +%Y%m%d).sql
```

```bash
# Восстановление из custom-формата (создать БД заранее при необходимости)
pg_restore -d mydb -j 4 /backup/mydb_20250101.dump

# Восстановление из SQL
psql -d mydb -f /backup/mydb.sql
```

### pg_basebackup (физический полный бэкап)

**Физический бэкап** — копия файлов кластера (data directory). **pg_basebackup** создаёт согласованный снимок с текущей позицией WAL; для PITR нужен ещё архив WAL.

```bash
# Полный бэкап в каталог (требуется настроенный доступ репликации в pg_hba.conf)
pg_basebackup -D /backup/base_$(date +%Y%m%d) -Ft -z -P -U replicator

# -D каталог, -Ft tar + сжатие, -z gzip, -P прогресс
```

Восстановление из base backup: остановить сервер, подставить файлы из бэкапа, при необходимости задать **recovery_target_time** и настроить **restore_command** для подтягивания WAL из архива, затем запустить — сервер дотянет WAL и выйдет из recovery.

### Full vs logical backup

| Тип | Инструмент | Плюсы | Минусы |
|-----|------------|-------|--------|
| **Logical** | pg_dump / pg_dumpall | Выборочное восстановление, перенос между версиями | Медленнее на больших объёмах, не «побайтовый» снимок |
| **Physical** | pg_basebackup + WAL архив | Быстрый полный restore, основа для PITR | Восстановление только той же мажорной версии, нужен архив WAL для PITR |

В production обычно комбинируют: **регулярный physical** (pg_basebackup или pgBackRest/wal-g) + **архив WAL** для PITR; при необходимости логический дамп для переноса или выборочного восстановления.

### Point-in-Time Recovery (PITR)

PITR: восстановление до произвольного момента в пределах архива WAL. Нужны: полный бэкап (base backup) + непрерывный архив WAL. В каталоге данных создаётся **recovery.spec** (или в PostgreSQL 12+ **postgresql.auto.conf** и **standby.signal** / настройки restore) с **recovery_target_time** и **restore_command**.

```ini
# recovery.spec (или в postgresql.conf при старых версиях)
recovery_target_time = '2025-01-15 14:30:00'
restore_command = 'cp /archive/wal/%f %p'
```

После восстановления до нужной точки сервер выходит из recovery и принимает подключения.

### Инструменты: pgBackRest, wal-g

- **pgBackRest** — полные и инкрементальные бэкапы, сжатие, параллелизм, дельта restore, архив WAL. Конфиг в отдельном файле; интеграция с S3, GCS, Azure.
- **wal-g** — архив WAL и бэкапы в облачное хранилище (S3 и др.); быстрый инкрементальный бэкап и restore. Удобен в облачных окружениях.

Оба подходят для production; выбор по экосистеме (облако, уже используемые инструменты) и требованиям к RPO/RTO.

### Практика: восстановить БД из бэкапа

1. **Логический:** создать пустую БД, выполнить `pg_restore -d mydb backup.dump` или `psql -d mydb -f backup.sql`.
2. **Физический + PITR:** остановить сервер; подставить данные из base backup; задать restore_command и при необходимости recovery_target_time; запустить сервер; дождаться выхода из recovery.
3. Проверить целостность и актуальность данных после восстановления.

---

## Best practices для production

| Область | Рекомендация |
|--------|--------------|
| **VACUUM** | Мониторить n_dead_tup и last_autovacuum; тюнить autovacuum по таблицам при необходимости; не отключать. |
| **WAL** | Достаточный объём диска под pg_wal и архив; мониторить место; алертить на частые checkpoints_req. |
| **Backup** | Регулярные полные бэкапы + архив WAL; хранить вне сервера; периодически проверять restore. |
| **PITR** | Тестировать восстановление до точки во времени в staging; документировать restore_command и процедуру. |

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Автоматические бэкапы + проверка restore** | Регулярный прогон восстановления в тестовом окружении. |
| **Мониторинг dead tuples и WAL** | Раннее выявление bloat и нехватки места. |
| **Архив WAL вне сервера** | Возможность PITR и восстановления при потере диска. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Отключить autovacuum** | Рост bloat и риск wraparound. | Настраивать пороги и cost, не отключать. |
| **Бэкапы только на тот же диск** | При отказе диска теряются и данные, и бэкап. | Копировать в другое хранилище (другой диск, объектное хранилище). |
| **Никогда не проверять restore** | В момент аварии процедура может не сработать. | Периодически восстанавливать из бэкапа в тестовый кластер. |

---

## Дополнительные материалы

- [PostgreSQL — Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [PostgreSQL — Write-Ahead Log](https://www.postgresql.org/docs/current/wal.html)
- [PostgreSQL — Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
- [pgBackRest](https://pgbackrest.org/)
- [wal-g](https://github.com/wal-g/wal-g)
- [1. База PostgreSQL](topic-1-basis.md) — MVCC, WAL, конфигурация
