# Apache Kafka
![Apache Kafka](./images/welcome.png)
---
Kafka — distributed log/message streaming платформа: позволяет организовывать поток событий, масштабировать обработку через consumer group’ы и добиваться отказоустойчивости благодаря replication.
---

## Материалы

- [**1. База**](topic-1-basis.md) — архитектура broker/cluster, partition’ы, replication (ISR), consumer group’ы и offset’ы; локальная практика на Docker Compose (ZooKeeper и KRaft).
- [**2. Установка и depлой**](topic-2-install-deploy.md) — bare metal/VM, Docker и production-deploy на Kubernetes (Strimzi), важные настройки listeners, advertised адресов, rack awareness и PVC.
- [**3. Хранение и производительность**](topic-3-storage-and-performance.md) — log-based storage, retention (time/size), compaction (cleanup.policy=compact) и DevOps-фокус по IO/page cache/JBOD vs RAID.
- [**4. Производительность и тюнинг**](topic-4-performance-tuning.md) — producer/broker/consumer параметры, анализ trade-offs и нагрузочное тестирование.
- [**5. Безопасность**](topic-5-security.md) — SASL аутентификация, TLS шифрование (в т.ч. между broker’ами) и ACL авторизация с secrets management (Vault/IAM).
- [**6. Мониторинг и алертинг**](topic-6-monitoring-alerting.md) — Prometheus/Grafana, JMX exporter, alert’ы по consumer lag, репликации, latency и disk usage.
- [**7. Kafka Connect**](topic-7-kafka-connect.md) — source/sink интеграции, Distributed/Standalone режимы, масштабирование workers и fault tolerance.
- [**8. CDC и Debezium**](topic-8-cdc-debezium.md) — Change Data Capture, logical replication/binlog, lag/backpressure и схема-эволюция.
- [**9. Отказоустойчивость и DR**](topic-9-fault-tolerance-and-dr.md) — broker down, network partition, disk full, recovery проверка и межкластерная репликация (MirrorMaker 2).
- [**10. Эксплуатация и best practices**](topic-10-operations-best-practices.md) — конвенции именования, strategy партиций, lifecycle данных, rolling updates, rebalancing и очистка устаревших потоков.