# 8. Лучшие практики

---
Свод правил уровня **Middle+**, которые стоит **явно принять в команде**: как **называть** метрики, когда брать **counter / gauge / histogram**, как вести себя с **лейблами** и когда выносить тяжёлый PromQL в **recording rules**. Это снижает шум, стоимость TSDB и время расследований.

---

## Именование (naming convention)

Ориентир — [**официальные практики Prometheus**](https://prometheus.io/docs/practices/naming/).

| Правило | Пример |
|---------|--------|
| Имя в **snake_case** | `http_requests_total`, `process_cpu_seconds_total` |
| **Суффикс `_total`** у counter’ов с накоплением | `orders_created_total` |
| **Базовые единицы** в имени | `_seconds`, `_bytes`, `_ratio` (не смешивать milliseconds в имени без причины) |
| **Gauge** без обязательного `_total` | `queue_depth`, `memory_usage_bytes` |

Хорошо:

```text
http_requests_total{method="GET", handler="/api/v1/orders", status="200"}
```

Плохо (смешение сущностей и «магические» префиксы):

```text
MyAppNumRequests{httpMethod="get"}   # несогласованный стиль
reqs                                   # нечитаемо в дашбордах команды
```

Комментарий: единый **префикс домена** (`payments_`, `corp_api_`) упрощает поиск и права на Federation/remote.

---

## Counter vs Gauge vs Histogram

| Тип | Когда использовать | Типичная ошибка |
|-----|-------------------|----------------|
| **Counter** | События только **вверх** (запросы, ошибки, байты отданные навсегда) | Применять `rate()` к gauge |
| **Gauge** | Значение **растёт и падает** (очередь, память процесса, активные соединения) | Хранить в gauge «всего за всё время» без смысла downtrend |
| **Histogram** | Распределение величин (латентность, размер ответа) — `_bucket`, `_sum`, `_count` | Слишком много/мало **buckets**; лейблы с высокой кардинальностью на bucket |

```promql
# Counter — всегда через rate/increase для «скорости»
rate(http_requests_total[5m])

# Gauge — можно смотреть «как есть» или сгладить
avg_over_time(queue_depth[5m])

# Histogram — квантиль только от _bucket + rate
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

---

## Гигиена лейблов (label hygiene)

| Делать | Не делать |
|--------|-----------|
| Низкая кардинальность: `method`, `status`, `deployment`, `region` | Уникальные на запрос: `user_id`, `session_id`, полный `url` |
| Стабильные значения `status` (`"200"`, `"5xx"` классом — осторожно) | Тысячи значений `handler` от генерируемых путей |
| Согласовать набор лейблов в **service level** | Каждый сервис со своим набором синонимов (`env` vs `environment`) |

Пример **антипаттерна** для кардинальности:

```text
http_requests_total{user_id="1847293"}  # миллионы рядов
redis_latency_seconds{key="session:abc123..."}
```

Лучше агрегировать **в приложении** или экспортироваться **без** ID:

```text
http_requests_total{route="/api/profile"}   # маршрут из белого списка
```

Production: периодически смотреть **top series** (`tsdb analyze` / отчёты Operator) после релизов.

---

## Recording rules для оптимизации

**Recording rules** заранее считают тяжёлые запросы и сохраняют результат как новую метрику — быстрее **алерты**, **дашборды** и снижается нагрузка на query.

### Пример группы

```yaml
groups:
  - name: recording.http
    interval: 1m
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job_route:http_requests:rate5m
        expr: sum by (job, handler) (rate(http_requests_total[5m]))

      - record: job:http_5xx:ratio5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (job) (rate(http_requests_total[5m]))
```

В дашборде и алерте дальше используете **`job:http_requests:rate5m`** вместо повторения длинного `rate(...)`.

Best practices:

- **`interval`** не меньше разумного для SLI (часто 30s–5m);
- имена записей в духе `level:metric:operations` — см. рекомендации в документации **Recording rules**;
- **не дублировать** сотни правил без отличий — review как у кода.

Проверка:

```bash
promtool check rules /etc/prometheus/recording/rules.yml
```

---

## Краткий чеклист команды

| # | Вопрос |
|---|--------|
| 1 | Есть ли **style guide** имён и лейблов в Confluence/Git? |
| 2 | Новые метрики проходят **review** на кардинальность? |
| 3 | Counter/gauge/histogram используются по таблице выше? |
| 4 | Топ-10 панелей Grafana используют **recording** где нужно? |
| 5 | Ломающее переименование метрик идёт через **migration** (двойная запись / алиасы)? |

---

## Дополнительные материалы

- [Naming metrics](https://prometheus.io/docs/practices/naming/)
- [Instrumentation](https://prometheus.io/docs/practices/instrumentation/)
- [Histograms and summaries](https://prometheus.io/docs/practices/histograms/)
- [Recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)

