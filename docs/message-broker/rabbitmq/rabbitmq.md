# RabbitMQ

---
RabbitMQ — message broker на базе AMQP: принимает сообщения от producer’ов, маршрутизирует их через exchange’и и доставляет в очереди (queues) для consumer’ов. Подходит для асинхронной обработки, разгрузки API, интеграций и событийных пайплайнов.

---

## Материалы

- [**1. База**](topic-1-basis.md) — архитектура RabbitMQ (producer/consumer, exchange/queue/binding), типы exchange, базовая практика запуска (Docker, Helm), основы надёжности (ack, durability, publisher confirms) и production best practices.
- [**2. Надёжность и delivery-паттерны**](topic-2-reliability-patterns.md) — ack/nack, confirms, prefetch, TTL, DLX/DLQ, retry‑паттерны и защитные настройки, чтобы «не терять сообщения» и не перегружать consumer’ов.
- [**3. Паттерны применения и архитектура**](topic-3-architecture-patterns.md) — work queues, pub/sub, RPC over RabbitMQ, event-driven подход, идемпотентность и практические схемы для production.
- [**4. Кластеры и High Availability**](topic-4-cluster-ha.md) — кластер RabbitMQ, classic vs quorum queues, network partitions, leader election и практики эксплуатации HA в production.
- [**5. Kubernetes и Production**](topic-5-kubernetes-production.md) — развёртывание в Kubernetes (StatefulSet/Operator), persistent storage, probes, ресурсы, доступ/безопасность и эксплуатационные best practices.
- [**6. Мониторинг и алертинг**](topic-6-monitoring-alerting.md) — метрики RabbitMQ, queue depth/backlog, consumer lag, message rates, Prometheus/Grafana и набор production‑алертов.
- [**7. Performance и тюнинг**](topic-7-performance-tuning.md) — throughput vs latency, connection/channel лимиты, prefetch tuning, lazy queues, нагрузочное тестирование и поиск bottlenecks.
- [**8. Troubleshooting**](topic-8-troubleshooting.md) — диагностика инцидентов: “сообщения зависли”, очередь растёт, consumer’ы падают, memory/disk alarms, network partition; чеклисты и команды.
- [**9. Security**](topic-9-security.md) — users/vhosts/permissions, TLS, auth backends и минимально необходимая “жёсткость” настроек для production.
- [**10. Сравнение и контекст**](topic-10-comparison-context.md) — Kafka, Redis (Pub/Sub/Streams) и когда RabbitMQ лучше/хуже; как выбрать инструмент под задачу.