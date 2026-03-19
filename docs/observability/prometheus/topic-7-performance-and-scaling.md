# 7. Производительность и масштабирование

---
На уровне **Middle+ → Senior** критично управлять **кардинальностью**, **ретенцией** и ростом **локального TSDB**: когда одного Prometheus уже мало — **remote write**, **федерация**, **шардинг** и внешние стеки (**Thanos**, **Grafana Mimir**, **VictoriaMetrics**). Ниже — сжато о проблемах, флагах/конфигурации и минимальных примерах «как в production».

---

## Cardinality — главный враг

**Cardinality** — число уникальных комбинаций лейблов (число активных time series). Рост = больше RAM, медленнее compact/query, риск OOM.

| Источник проблемы | Что делать |
|-------------------|------------|
| Лейблы с неограниченной вариативностью (`user_id`, `url` полный путь) | Убрать из экспозиции или агрегировать на стороне приложения |
| Высокий churn Pod’ов в K8s | Группировать в PromQL по `deployment`, `namespace`; **`metric_relabel_configs`** для дропа избыточных лейблов |
| Слишком много scrape targets | Шардирование по командам/кластерам |

Пример **отбрасывания** дорогой метрики на запись в long-term storage:

```yaml
remote_write:
  - url: https://remote.example/api/v1/write
    write_relabel_configs:
      - source_labels: [__name__]
        regex: "debug_.*|go_gc_.*"
        action: drop
```

Комментарий: **не дропайте** то, без чего ломаются алерты; изменения проходят **review** и замер **до/после** (`prometheus_tsdb_head_series`).

---

## Retention и локальный TSDB

| Параметр | Смысл |
|----------|--------|
| **`storage.tsdb.retention.time`** | Как долго держать данные локально (например `15d`) |
| **`storage.tsdb.retention.size`** | Потолок размера блоков на диске |

Флаги при запуске (пример):

```bash
prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB \
  --storage.tsdb.wal-compression
```

Best practice: локальный Prometheus — «**горячее**» окно для оперативных запросов и алертов; **долгий** архив — через remote backend или sidecar-стек.

---

## Remote write

**Remote write** асинхронно отправляет сэмплы во внешнее хранилище (VictoriaMetrics, Mimir, Cortex-совместимое и т.д.).

```yaml
remote_write:
  - url: https://vminsert.internal:8480/insert/0/prometheus/api/v1/write
    queue_config:
      max_samples_per_send: 10000
      batch_send_deadline: 5s
      max_shards: 50
    basic_auth:
      username: prometheus
      password_file: /etc/prometheus/secrets/remote_pass
```

Production:

- **TLS**, лимиты **queue_config** под пропускную ссылку;
- мониторинг **`prometheus_remote_storage_*`**;
- при сетевых сбоях растёт **WAL** — следить за диском.

---

## Federation

**Федерация** — «верхний» Prometheus **скрейпит** `/federate` у других и подтягивает выбранные серии (часто агрегаты).

```yaml
scrape_configs:
  - job_name: federate-dc1
    scrape_interval: 30s
    honor_labels: true
    metrics_path: /federate
    params:
      match[]:
        - '{job="kubernetes-nodes"}'
        - 'up'
    static_configs:
      - targets:
          - prometheus-dc1.internal:9090
```

Best practice: федерация **не заменяет** полноценный global query layer; выгружайте **ограниченный** набор `match[]`, иначе снова кардинальность.

---

## Sharding (шардирование)

Идеи без смены продукта:

- несколько **Prometheus** с непересекающимися **targets** (по namespace, по региону, `hashmod` в **hashmod** relabel);
- **external_labels** на каждом шарде (`replica`, `cluster`), чтобы в **remote storage** не смешивать идентичность серий.

Фрагмент идеи с **`hashmod`** (упрощённо):

```yaml
relabel_configs:
  - source_labels: [__address__]
    modulus: 2
    target_label: __tmp_shard
    action: hashmod
  - source_labels: [__tmp_shard]
    regex: "^1$"
    action: keep
```

Комментарий: реальный дизайн шардов зависит от SD; часто проще **разные scrape job’ы** по `role: metrics-team-a`.

---

## Инструменты долгого хранения и «глобального» запроса

| Проект | Коротко |
|--------|---------|
| **Thanos** | Sidecar к Prometheus + **Query** + object storage; дедупликация реплик |
| **Grafana Mimir** | Горизонтально масштабируемое хранилище, Remote write, совместимость PromQL |
| **VictoriaMetrics** | Высокая сжимаемость, **single-binary** или кластер (**vminsert/vmselect/vmstorage**) |

Best practice: выбор по **операционной зрелости команды**, SLO на query latency и стоимость object storage, а не по «модному названию».

---

## Практика: включить remote write

1. Поднять приёмник (**VictoriaMetrics** или **Mimir** по гайду вендора).
2. Создать пользователя/токен, открыть **только** сетевой путь от Prometheus.
3. Добавить блок **`remote_write`** в конфиг, начать с **одного** окружения (staging).
4. Сравнить **серии** и **lag** (`prometheus_remote_storage_highest_timestamp_in_seconds` vs `time()`).
5. Подключить **Grafana** вторым datasource к долгому хранилищу (или через Query frontend).

---

## Практика: Thanos или VictoriaMetrics (ориентир)

### VictoriaMetrics (минимальный контур)

Частый путь: **single-node** для старта или **`vmagent`** → **cluster**.

```bash
# Учебный single-node (параметры ретенции см. актуальную документацию VM)
docker run -d --name victoriametrics -p 8428:8428 \
  victoriametrics/victoria-metrics:latest \
  -storageDataPath=/victoria-data
# remote_write URL для Prometheus: http://<host>:8428/api/v1/write
```

### Thanos (идея)

На каждом Prometheus — **Thanos Sidecar** (доступ к TSDB), **Thanos Query** агрегирует Store/Sidecar; исторические блоки выгружаются в **S3-совместимое** хранилище. Детали — в документации Thanos; в production обязательны **compact**, политика **bucket**, права IAM.

---

## Сводный checklist

| # | Вопрос |
|---|--------|
| 1 | Измерена ли **head series** и топ «тяжёлых» метрик? |
| 2 | Согласованы **retention** локально и в remote? |
| 3 | Есть ли алерты на **remote write failures** и **WAL**? |
| 4 | Федерация не тянет ли «всё подряд»? |
| 5 | Для global view выбран ли один путь (Thanos / Mimir / VM), а не три параллельно? |

---

## Дополнительные материалы

- [Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Remote write tuning](https://prometheus.io/docs/practices/remote_write/)
- [Thanos](https://thanos.io/)
- [Grafana Mimir](https://grafana.com/docs/mimir/latest/)
- [VictoriaMetrics](https://docs.victoriametrics.com/)

