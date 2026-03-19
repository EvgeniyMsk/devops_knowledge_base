# MongoDB

![MongoDB[200]](./mongodb/images/welcome.jpg)
**MongoDB** — документная **NoSQL** СУБД: **BSON**-документы, **коллекции**, **индексы**, горизонтальное масштабирование (**replica set**, **sharded cluster**) в enterprise-контурах. В материалах ниже — от основ до эксплуатации и production-паттернов.

## Материалы

- [**1. База**](mongodb/topic-1-basis.md) — NoSQL и зачем MongoDB, BSON, структура database/collection/document, CRUD в mongosh, типы индексов (single, compound, TTL, text), `explain()`, Docker для практики.
- [**2. Архитектура**](mongodb/topic-2-replica-architecture.md) — replica set (primary/secondary/arbiter), election и падение primary, write/read concern, WiredTiger (cache, сжатие, journaling), лабораторная с тремя узлами и симуляцией сбоев.
- [**3. Шардирование**](mongodb/topic-3-sharding.md) — `mongos`, config servers, шарды; ranged vs hashed shard key; hot shard и перекос; команды `sh.enableSharding`, `shardCollection`, `sh.status`, практика кластера и проверка распределения.
- [**4. Операционка**](mongodb/topic-4-operations.md) — Docker/Kubernetes и StatefulSet, PV; `mongod.conf`, `keyFile`, TLS; `mongodump`/`mongorestore`, снапшоты, PITR и oplog; восстановление при **partial failure** и чеклист DR.
- [**5. Мониторинг**](mongodb/topic-5-monitoring.md) — Prometheus, Grafana, MongoDB Exporter; метрики opcounters, connections, replication lag, кэш, диск; дашборд и алерты (lag, отсутствие PRIMARY).
- [**6. Troubleshooting**](mongodb/topic-6-troubleshooting.md) — lag, медленные запросы, конкуренция и диск; `mongostat`, `mongotop`, profiler, логи; `explain`, `currentOp`, практика с нагрузкой и индексами.
- [**7. Безопасность**](mongodb/topic-7-security.md) — SCRAM, роли и кастомные роли, TLS, сетевая изоляция; создание пользователей, строка подключения, практика включения auth.
- [**8. Production-практики**](mongodb/topic-8-production-practices.md) — capacity planning и диск, бэкапы (RPO/RTO), rolling updates и FCV; антипаттерны (индексы, shard key, один узел, бэкапы без restore).
- [**9. Продвинутые темы**](mongodb/topic-9-advanced.md) — Change Streams; транзакции; мультирегион; Atlas vs self-hosted; скетчи Ansible и Terraform.
- [**10. Интеграционная практика**](mongodb/topic-10-integrated-lab.md) — production-like стенд: RS, 2 шарда, Prometheus/Grafana, бэкапы, K8s; сценарии падения узла, деградации диска, нагрузки и восстановления.

