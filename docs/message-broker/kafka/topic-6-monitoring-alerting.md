# 6. Мониторинг и алертинг

---

Kafka в production почти невозможно поддерживать без мониторинга. В этой теме — что именно измерять и как построить алертинг так, чтобы MTTR был минимальным, а false positives — контролируемыми.

Цели:

- увидеть деградацию до инцидента,
- быстро понять “почему” по симптомам (lag растёт, брокер упал, репликации не хватает, latency взлетела, диск заполнился).

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Prometheus** | Система сбора метрик и хранения time-series, используется как backend для алертов и dashboard’ов. |
| **Grafana** | Визуализация метрик (dashboards) поверх Prometheus. |
| **JMX Exporter** | Экспортер метрик из JVM/JMX в формате, который понимает Prometheus. |
| **Consumer lag** | Разница между “самым новым” смещением/записями и тем, до чего дошёл consumer group. |
| **Under-replicated partitions** | Партиции, у которых недостаточно реплик в сравнении с требуемыми in-sync условиями. |
| **Request latency** | Задержка обработки запросов Kafka (produce/fetch) на уровне broker/клиента. |
| **Disk usage** | Занятость диска broker’а (и/или соответствующих volumes). |

---

## Что мониторить в первую очередь (симптомы из production)

### 1) Consumer lag растёт

Почему это важно:

- lag может означать проблему в consumer’ах (slow processing),
- или недостаток параллелизма,
- или внешнюю зависимость (БД/HTTP) “тянет” вниз throughput.

Сколько держать “тишину”:

- алерт должен реагировать на sustained рост (например, 5–10 минут), а не на единичные всплески.

### 2) Under-replicated partitions

Почему это важно:

- репликации могут отставать из-за диска, сети или проблем с ISR,
- это повышает риск потери доступности при следующем отказе.

Production подход:

- severity зависит от того, насколько критична политика in-sync (минимально необходимое число реплик).

### 3) Request latency

Почему важно:

- рост latency обычно первичен (до полного падения),
- часто связано с диском (IO bottleneck), очередями или GC/ресурсами.

### 4) Disk usage

Почему важно:

- при переполнении диска broker начинает деградировать,
- задержки очистки (retention/compaction) часто приводят к “неожиданному” fill-up.

---

## Рекомендованный набор dashboard’ов

1) **Kafka overview**: health, request latency (p95/p99), produce/fetch rates, error rates.
2) **Consumer lag**: lag по consumer group’ам + корреляция с throughput.
3) **Replication health**: under-replicated partitions + динамика ISR.
4) **Disk & storage**: disk usage, disk IO, filesystem errors, заполненность volumes.

Best practice:

- один dashboard на “следующую диагностику”: увидели симптом — сразу переходите к panel’ам для причины.

---

## Алерты: базовая стратегия (чтобы не утонуть)

Сделайте алертинг в три слоя:

- **warning**: ранний сигнал,
- **critical**: действия немедленно,
- **page only**: только при реальном риске инцидента (broker down, критическая репликация, заполнение диска).

Best practices:

- используйте `for:` (срабатывание после устойчивого периода), например `for: 10m`,
- вводите “пороговые окна”, чтобы избегать одиночных всплесков,
- добавляйте `runbook` ссылку/текст для on-call (что проверить за 2–5 минут).

---

## Пример: Prometheus scrape и rules (шаблон)

Примечание: точные имена метрик зависят от того, какой exporter вы используете (Strimzi metrics, JMX exporter, kafka-exporter и т.д.). Ниже — шаблон с понятными подстановками.

### Пример `prometheus.yml` scrape

```yaml
scrape_configs:
  - job_name: kafka
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: kafka
    metrics_path: /metrics
    scheme: http
```

### Пример rules (consumer lag / under-replicated / latency / disk)

```yaml
groups:
  - name: kafka.rules
    rules:
      - alert: ConsumerLagGrowing
        expr: increase(kafka_consumer_group_lag[10m]) > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Consumer lag растёт"
          description: "Проверьте consumer’ы, throughput и downstream зависимости."

      - alert: UnderReplicatedPartitions
        expr: kafka_under_replicated_partitions > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Недостаточно реплик"
          description: "Проверьте диски/сеть broker’ов и ISR."

      - alert: RequestLatencyHigh
        expr: histogram_quantile(0.99, rate(kafka_request_latency_seconds_bucket[5m])) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Latency p99 выросла"
          description: "Смотрите корреляцию с IO/GC и очередями."

      - alert: DiskUsageHigh
        expr: kafka_broker_disk_usage_ratio > 0.85
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Диск почти заполнен"
          description: "Проверьте retention/compaction, объемы и ошибки очистки."
```

Комментарий:

- пороги (`> 0.5`, `0.85`) ставьте исходя из базовой линии production (baseline),
- используйте единый стиль порогов для предупреждений и critical.

---

## Практика-триаж: когда алерт прилетел

Минимальный набор команд (быстро подтвердить симптом):

```bash
# consumer lag в разрезе consumer group’а
kafka-consumer-groups.sh \
  --bootstrap-server kafka-0.kafka:9092 \
  --describe \
  --group orders-worker

# события по broker’у/подам (когда broker упал или рестартится)
kubectl -n kafka get events --sort-by='.lastTimestamp'
```

Best practice:

- всегда связывайте алерт с конкретным объектом: consumer group / broker / volume,
- затем идите в dashboard за корреляцией (latency, disk usage, replication).

---

## Production checklist

| Что сделать | Зачем |
|--------------|-------|
| Consumer lag алертить по “устойчивому росту” | меньше false positives |
| Under-replicated и ISR-симптомы в critical | высокий риск следующего отказа |
| Request latency p95/p99 + корреляция с диском | быстро локализует bottleneck |
| Disk usage с threshold и runbook | предотвращает деградацию из-за переполнения |
| Dashboard’ы по этапу диагностики | MTTR уменьшается системно |

