# 4. Производительность и диагностика: запросы, EXPLAIN, индексы

**Цель:** уметь диагностировать производительность запросов: читать EXPLAIN и EXPLAIN ANALYZE, различать sequential scan и index scan; использовать pg_stat_statements и auto_explain; понимать типы индексов (B-tree, GIN, GiST) и их влияние на запись, REINDEX и bloat индексов. В разделе — примеры кода и SQL с комментариями и best practices для production.

---

## 10. Query performance

### EXPLAIN и EXPLAIN ANALYZE

**EXPLAIN** показывает план выполнения запроса (без запуска); **EXPLAIN ANALYZE** выполняет запрос и добавляет фактические строки, время и I/O. Для диагностики медленных запросов всегда смотреть **EXPLAIN (ANALYZE, BUFFERS)** — видно, сколько буферов прочитано с диска (shared hit/read).

```sql
-- Базовый план (оценки планировщика)
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- План с фактическим выполнением и буферами (ключевой вариант для отладки)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 123;
```

Типичный вывод:
- **Seq Scan** — полный перебор таблицы; при больших таблицах часто признак отсутствия подходящего индекса или того, что планировщик считает полный скан дешевле.
- **Index Scan / Index Only Scan** — доступ по индексу; Index Only Scan возможен, если все нужные столбцы есть в индексе (см. visibility map).
- **Rows** — оценка (EXPLAIN) или факт (ANALYZE); большое расхождение между estimate и actual говорит о устаревшей статистике — нужен **ANALYZE** таблицы.
- **Buffers: shared hit** — из кэша, **shared read** — с диска; много read — возможный признак нехватки shared_buffers или холодного кэша.

### Sequential Scan vs Index Scan

- **Sequential Scan (Seq Scan):** читает все страницы таблицы по порядку. Эффективен, когда нужна большая доля строк (например, > 5–10%) или таблица маленькая; планировщик может выбрать его, если индекс не выгоден.
- **Index Scan:** по индексу находит указатели на строки (heap tuples), затем читает нужные страницы таблицы. Выгоден при малой доле строк и селективных условиях (WHERE, JOIN). **Index Only Scan** — если все столбцы в индексе и видимость есть в visibility map, таблицу не читают.

Если запрос медленный и в плане **Seq Scan по большой таблице** при селективном условии — проверить: есть ли подходящий индекс; актуальна ли статистика (ANALYZE); не занижены ли оценки из-за неверной статистики.

```sql
-- Пример: после добавления индекса планировщик может перейти на Index Scan
CREATE INDEX idx_orders_user_id ON orders (user_id);
ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 123;
```

### Когда звать разработчика

DevOps может: находить тяжёлые запросы (pg_stat_statements), смотреть планы, проверять индексы и статистику, настраивать autovacuum и параметры. К разработчику обращаться, когда: нужна смена логики запроса (переписать JOIN, убрать N+1, изменить тип запроса); нужны новые индексы или изменение схемы; выявлена проблема в коде приложения (частые запросы, неоптимальный ORM). Совместно: решение по индексам с учётом нагрузки на запись.

### pg_stat_statements

Расширение **pg_stat_statements** собирает агрегированную статистику по нормализованному тексту запроса: число вызовов, суммарное и среднее время, rows, shared_blks_hit/read. Позволяет найти самые затратные запросы по времени и по I/O.

```sql
-- Установка (один раз от суперпользователя)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- В postgresql.conf должно быть: shared_preload_libraries = 'pg_stat_statements'
-- И pg_stat_statements.max, pg_stat_statements.track — при необходимости
```

```sql
-- Топ запросов по суммарному времени
SELECT
  round(total_exec_time::numeric, 2) AS total_ms,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  rows,
  left(query, 80) AS query_short
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 15;
```

```sql
-- Топ по числу обращений к диску (shared_blks_read)
SELECT
  calls,
  shared_blks_hit,
  shared_blks_read,
  round(total_exec_time::numeric, 2) AS total_ms,
  left(query, 100) AS query_short
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 10;
```

Сброс счётчиков: **pg_stat_statements_reset()** (при необходимости — после деплоя или для чистого замера).

### auto_explain

**auto_explain** — модуль, который автоматически логирует планы «долгих» запросов (выше заданного порога по времени). Удобен для сбора планов в production без ручного EXPLAIN.

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_format = text
```

После reload запросы, выполняющиеся дольше 1000 ms, будут логироваться с планом (ANALYZE, BUFFERS). Порог задавать под нагрузку (например, 500–2000 ms).

---

## 11. Индексы и bloat

### B-tree, GIN, GiST: когда что использовать

| Тип | Назначение | Примеры использования |
|-----|------------|------------------------|
| **B-tree** (по умолчанию) | Сравнения, сортировка, диапазоны, UNIQUE. | WHERE id = ..., WHERE created_at > ..., ORDER BY, JOIN по ключу. |
| **GIN** | Множества, полнотекстовый поиск, массивы, jsonb. | @>, ? , полнотекст, containment. |
| **GiST** | Геоданные, диапазоны, полнотекст (альтернатива GIN). | Операторы для geometric, range, full text; часто для «близости» и overlap. |

Для обычных равенств и диапазонов — **B-tree**. Для полей с множествами, JSONB, full text — **GIN** (или GiST по необходимости). Выбор типа индекса влияет на скорость запросов и на стоимость записи (индексы обновляются при INSERT/UPDATE/DELETE).

```sql
-- B-tree: по умолчанию
CREATE INDEX idx_orders_created ON orders (created_at);
CREATE UNIQUE INDEX idx_users_email ON users (email);

-- GIN для jsonb (например, поиск по ключу в JSON)
CREATE INDEX idx_events_properties ON events USING GIN (properties jsonb_path_ops);

-- GIN для полнотекстового поиска
CREATE INDEX idx_articles_fts ON articles USING GIN (to_tsvector('russian', title || ' ' || body));
```

### REINDEX vs VACUUM FULL

- **REINDEX** — перестроить индексы (все индексы таблицы или один индекс). Устраняет bloat индекса, освобождает место; блокирует запись в индекс на время перестроения (в PostgreSQL 12+ **REINDEX CONCURRENTLY** — без эксклюзивной блокировки таблицы).
- **VACUUM FULL** — перезаписать саму таблицу, затем перестроить индексы. Требует эксклюзивной блокировки таблицы; применяется при сильном bloat таблицы, не только индекса.

При bloat только индекса — **REINDEX** (лучше **REINDEX INDEX CONCURRENTLY** в production). При bloat таблицы и необходимости вернуть место на диске — **VACUUM FULL** в окно обслуживания.

```sql
-- Перестроить один индекс без длительной блокировки таблицы (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_orders_created;

-- Перестроить все индексы таблицы (блокирует запись в таблицу при обычном REINDEX)
-- REINDEX TABLE CONCURRENTLY orders;
```

### Влияние индексов на write-нагрузку

Каждый индекс при **INSERT** и **UPDATE** (затрагивающем проиндексированные столбцы) и при **DELETE** требует обновления. Больше индексов — выше стоимость записи и больше WAL. В write-heavy нагрузке лишние индексы замедляют вставки; удалять неиспользуемые индексы (проверять через **pg_stat_user_indexes** — idx_scan).

```sql
-- Индексы без обращений (кандидаты на удаление при необходимости)
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey';
```

Не удалять «на глаз» — убедиться, что индекс не нужен для редких критичных запросов или ограничений (UNIQUE).

---

## Best practices для production

| Область | Рекомендация |
|--------|---------------|
| **Диагностика запросов** | Всегда смотреть EXPLAIN (ANALYZE, BUFFERS); при расхождении estimate/actual — ANALYZE таблицы. |
| **pg_stat_statements** | Включить в shared_preload_libraries; регулярно анализировать топ по time и по I/O. |
| **auto_explain** | Включить с разумным log_min_duration; не ставить слишком низкий порог (шум в логах). |
| **Индексы** | Добавлять по результатам анализа планов и pg_stat_statements; удалять неиспользуемые (после проверки). |
| **REINDEX** | В production предпочитать REINDEX INDEX CONCURRENTLY; планировать в окно обслуживания при REINDEX TABLE. |

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **EXPLAIN ANALYZE перед изменением индексов** | Сравнить план до и после; убедиться в улучшении. |
| **Мониторинг топ запросов** | pg_stat_statements + дашборды; реагировать на деградацию. |
| **REINDEX CONCURRENTLY для bloat индексов** | Без длительной блокировки записи. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Индекс на каждую колонку** | Рост стоимости записи и bloat; часть индексов не используется. | Индексы только под реальные запросы; проверять idx_scan. |
| **Игнорировать Seq Scan на больших таблицах** | Медленные запросы при селективном условии. | Проверить наличие индекса и статистику; при необходимости добавить индекс. |
| **VACUUM FULL в пик нагрузки** | Долгая эксклюзивная блокировка. | Выполнять в окно обслуживания; по возможности решать bloat через обычный VACUUM и настройки autovacuum. |

---

## Дополнительные материалы

- [PostgreSQL — EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- [PostgreSQL — pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [PostgreSQL — auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)
- [PostgreSQL — Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [PostgreSQL — REINDEX](https://www.postgresql.org/docs/current/sql-reindex.html)
- [2. Эксплуатация и стабильность](topic-2-operations-stability.md) — VACUUM, bloat таблиц
