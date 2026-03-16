# PostgreSQL
![PostgreSQL[200]](./images/welcome.webp)

## Разделы

- [**1. База: архитектура, конфигурация и безопасность**](postgresql/topic-1-basis.md) — процессная модель, shared buffers, WAL, MVCC, TOAST/FSM/visibility map; postgresql.conf, pg_hba.conf, ключевые параметры (restart vs reload); роли, GRANT/REVOKE, pg_hba, SSL/TLS, SCRAM, минимальные привилегии и сервисные аккаунты.

- [**2. Эксплуатация и стабильность**](postgresql/topic-2-operations-stability.md) — VACUUM и Autovacuum (dead tuples, wraparound, bloat), WAL и checkpoints (fsync, synchronous_commit, мониторинг), Backup & Restore (pg_dump, pg_basebackup, PITR, pgBackRest, wal-g).

- [**3. High Availability и масштабирование**](postgresql/topic-3-ha-scaling.md) — streaming replication (sync/async), replication slots, lag; failover и HA (Patroni, repmgr, облако); connection pooling (PgBouncer, Pgpool), устранение «too many connections».

- [**4. Производительность и диагностика**](postgresql/topic-4-performance-diagnostics.md) — EXPLAIN/EXPLAIN ANALYZE, sequential vs index scan, pg_stat_statements, auto_explain; индексы (B-tree, GIN, GiST), REINDEX vs VACUUM FULL, влияние индексов на запись.

- [**5. Мониторинг и алертинг**](postgresql/topic-5-monitoring-alerting.md) — метрики (connections, replication lag, vacuum, disk, WAL, locks); postgres_exporter, Prometheus, Grafana; алерты без шума (lag, autovacuum, диск, idle in transaction).

- [**6. Автоматизация и DevOps**](postgresql/topic-6-automation-devops.md) — PostgreSQL в Docker/Kubernetes (StatefulSet ≠ HA, persistent volumes), операторы (Zalando, CrunchyData); IaC (Ansible, Terraform для managed DB), миграции (Flyway, Liquibase).

*(Дальнейшие разделы будут добавляться по мере подготовки материалов.)*
