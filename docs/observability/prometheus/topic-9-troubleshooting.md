# 9. Устранение неполадок

---
На уровне **Middle+** нужно уметь быстро понять, почему **цель не скрейпится**, почему на графиках **«дыры»**, почему **PromQL тормозит**, и откуда берутся **высокая память** и **кардинальность**. Ниже — типовые причины, метрики и команды, плюс учебные сценарии «сломать exporter / target / сеть».

---

## Prometheus не скрейпит target

### Куда смотреть

1. **UI** → *Status → Targets* — состояние **DOWN**, последняя ошибка (`Last scrape`, `Error`).
2. Метрика **`up{job="..."} == 0`** в момент времени.
3. Логи Prometheus (`--log.level=debug` — только временно на стенде).

### Типовые причины

| Симптом | Что проверить |
|---------|----------------|
| `connection refused` | Порт, firewall, Pod не слушает, не тот `target` после relabel |
| `context deadline exceeded` | **`scrape_timeout`** меньше времени ответа; перегружен exporter |
| `server returned HTTP status 404` | Неверный **`metrics_path`** (не `/metrics`) |
| TLS / certificate errors | **`scheme: https`**, CA, `insecure_skip_verify` только временно |
| Target пропал из списка | **Service discovery** пустой (лейблы ServiceMonitor, namespace) |
| Все таргеты job’а DOWN | Prometheus **не достигает** сеть namespace (Policy, DNS) |

### Проверка сети с той же «точки зрения», что у Prometheus

```bash
# Пример: из Pod Prometheus (имя подставьте свои)
kubectl -n monitoring exec -it prometheus-0 -c prometheus -- \
  wget -qO- "http://node-exporter.monitoring.svc:9100/metrics" | head
```

```bash
# TLS-цель
curl -v --cacert ca.pem https://app.internal:8443/metrics
```

---

## Данные «дырявые» (gaps в рядах)

| Причина | Детали |
|---------|--------|
| **Нестабильный scrape** | Цель перезапускается, `up` мигает |
| **Prometheus перегружен** | Очередь scrape, не успевает опросить в срок |
| **Clock skew** | Разное время на Prometheus и targets — реже, но проверяйте **NTP** |
| **Compaction / restart** | Короткий разрыв при рестарте или при проблемах с диском |
| **`rate()` на дырах** | В PromQL «провалы» видны как просадки — отличать от реального падения трафика |

Полезные метрики:

```promql
prometheus_tsdb_head_samples_appended_total
scrape_duration_seconds{job="..."}
```

Диагностика: сравнить `scrape_interval`, **интервал** панели Grafana и длительность `scrape_duration_seconds`.

---

## Медленные запросы PromQL

| Причина | Что сделать |
|---------|-------------|
| Огромный **диапазон** времени и тяжёлые функции | Сужать окно; вынести в **recording rule** |
| Высокая **кардинальность** внутри `sum by (...)` | Упростить лейблы; дроп на ingest |
| Много **subquery** и вложенности | Упростить; кэшировать промежуточный результат |
| Одновременно много тяжёлых дашбордов | **Query concurrency** и отдельный read path (Thanos Query / Mimir) |

Проверка нагрузки API (ориентир по метрикам):

```promql
prometheus_engine_queries
prometheus_engine_query_duration_seconds
```

Best practice: включить **query log** (с осторожностью к диску) на стенде и найти самые долгие запросы.

---

## Высокое потребление памяти

| Источник | Комментарий |
|----------|-------------|
| **Head block** TSDB | Много активных серий и сэмплов до compact |
| **Запросы** | Длинные range query держат данные в памяти |
| **Кардинальность** | Линейно бьёт по RAM и скорости compact |

Ориентиры по метрикам:

```promql
process_resident_memory_bytes{job="prometheus"}

prometheus_tsdb_head_series
prometheus_tsdb_symbol_table_size_bytes
```

Действия: снизить кардинальность, **`/api/v1/tsdb/delete_series`** (опасно, по runbook), уменьшить retention, масштабировать (шард), remote write + короткая локальная retention.

---

## Высокая кардинальность

1. **Grafana Explore** / UI Prometheus: какие `__name__` занимают больше всего внимания операторов (без тяжёлых запросов на огромном проде).
2. **Точный** офлайн-разбор: **`promtool tsdb analyze <path-to-tsdb>`** на **копии** тома TSDB, не на живом каталоге во время работы.
3. Ищите антипаттерны в лейблах: `trace_id`, полный `path`, неограниченный `user_id`.

Best practice: после каждого релиза сервиса с новыми метриками — **ревью** instrumentation.

---

## Практика: искусственно сломать (лаборатория)

Выполняйте **только** на тестовом кластере.

### 1. «Сломать» exporter

- остановить процесс `node_exporter` или масштабировать Deployment в 0;
- ожидание: target **DOWN**, алерт на `up == 0`;
- починка: вернуть реплики, проверить *Targets*.

### 2. «Сломать» target в конфиге

- опечатка в **`static_configs.targets`** или неверный порт в `Service`;
- симптом: `connection refused` / `no such host`.

### 3. «Сломать» сеть

- **NetworkPolicy** «deny all» из namespace Prometheus к приложению;
- симптом: таймауты scrape;
- учение: по логам CNI/Policy искать правило.

Документируйте: что увидели в **Last Error**, какой **метрикой** подтвердили восстановление.

---

## Краткий runbook (чеклист)

| Шаг | Действие |
|-----|----------|
| 1 | *Targets* / `up` — локализовать job и instance |
| 2 | Сеть + DNS + порт + путь `/metrics` |
| 3 | TLS и права scrape (mTLS, Bearer) |
| 4 | SD и relabel (не «выпилил» ли `keep/drop`) |
| 5 | Ресурс Prometheus: память, диск WAL, медленные запросы |

---

## Дополнительные материалы

- [Query troubleshooting](https://prometheus.io/docs/prometheus/latest/querying/troubleshooting/)
- [TSDB — operational aspects](https://prometheus.io/docs/prometheus/latest/storage/#operational-aspects)
- [Query log](https://prometheus.io/docs/guides/query-log/)

