# 10. Production-кейсы

---
Финальный этап — собрать **как в жизни**: мониторинг **микросервисов**, договорённости по **SLI/SLO** (латентность, ошибки, доступность) и опора на **Golden Signals** — **latency**, **traffic**, **errors**, **saturation**. Ниже — связка метрик, PromQL и практик, которые реально встречаются в production.

---

## Мониторинг микросервисов

| Подход | Смысл |
|--------|--------|
| **RED** (Rate, Errors, Duration) | Узкий набор для **синхронных** API: RPS, доля ошибок, задержка |
| **USE** (Utilization, Saturation, Errors) | Чаще для **узлов** и ресурсов (CPU, диск, сеть) |
| **Единый стиль** метрик | Одинаковые имена (`http_requests_total`, `http_request_duration_seconds_bucket`) во всех сервисах команды |

Best practices:

- **SLO на уровень сервиса** (или критичного API), а не «один дашборд на весь кластер»;
- **корреляция**: trace id в логах + те же лейблы `service`, `version` в метриках (без высокой кардинальности);
- алерты **пользовательских** симптомов (рост ошибок, p99), а не только «вырос CPU» без контекста.

---

## SLI / SLO: латентность, error rate, availability

| Понятие | Определение |
|---------|--------------|
| **SLI** | Измеримый индикатор качества (доля успешных запросов < 300 ms) |
| **SLO** | Цель по SLI за период (99.9% запросов быстрее 300 ms **ежемесячно**) |
| **Error budget** | Допустимая доля «плохих» событий до нарушения SLO |

### Примеры SLI в PromQL

**Доля успешных HTTP-запросов** (класс `2xx/3xx` как «успех» — уточните под контракт):

```promql
sum(rate(http_requests_total{status=~"2.."}[30m]))
/
sum(rate(http_requests_total[30m]))
```

**Доступность с точки зрения blackbox** (доля успешных проб):

```promql
avg_over_time(probe_success{job="api-health"}[1h])
```

**Латентность p99** (гистограмма HTTP):

```promql
histogram_quantile(
  0.99,
  sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
)
```

Комментарий: **окно** `[30m]` / `for` в алертах согласуйте с **multi-burn-rate** подходом (Google SRE) или внутренним стандартом.

### Recording rules под SLO (фрагмент)

```yaml
groups:
  - name: slo.api
    interval: 1m
    rules:
      - record: job:api:availability30m
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[30m])) by (job)
          / sum(rate(http_requests_total[30m])) by (job)

      - record: job:api:error_ratio5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          / sum(rate(http_requests_total[5m])) by (job)
```

Best practice: **задокументировать** SLI в одном месте (wiki) и не менять семантику `status` без миграции дашбордов.

---

## Golden Signals (четыре столпа)

| Сигнал | Что измерять | Пример PromQL / источник |
|--------|----------------|---------------------------|
| **Latency** | Время ответа (p50/p95/p99) | `histogram_quantile` по `_bucket` |
| **Traffic** | Нагрузка (RPS, байты/с) | `sum(rate(http_requests_total[5m]))` |
| **Errors** | Ошибки приложения и протокола | Доля `5xx`, `rate(errors_total)` |
| **Saturation** | Насколько сервис/узел «уперся» | Очередь, CPU throttle, `container_cpu_cfs_throttled_seconds_total`, заполнение пула |

### Latency

```promql
histogram_quantile(
  0.95,
  sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
)
```

### Traffic

```promql
sum by (job, handler) (rate(http_requests_total[5m]))
```

### Errors

```promql
sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (job) (rate(http_requests_total[5m]))
```

### Saturation (пример: CPU throttle в Kubernetes)

```promql
sum by (pod, namespace) (
  rate(container_cpu_cfs_throttled_seconds_total{container!=""}[5m])
)
/
sum by (pod, namespace) (
  rate(container_cpu_usage_seconds_total{container!=""}[5m])
)
```

Комментарий: для **брокеров** и **БД** saturation часто — **lag очереди**, **disk almost full**, **connection pool** — добавляйте **свои** gauge/counter по месту.

---

## Мини-чеклист «как у зрелой команды»

| # | Пункт |
|---|--------|
| 1 | На каждый критичный сервис — **RED** + **SLO** в Confluence/Git |
| 2 | Дашборд «золотых сигналов» + ссылки **runbook** |
| 3 | Алерты по **burn rate** / budget, а не по «всем подряд пикам» |
| 4 | Релизы смотрят на **регрессию** error_ratio и latency |
| 5 | Исключения из SLO (maintenance) оформлены и **не ломают** статистику молча |

---

## Дополнительные материалы

- [Google SRE — Monitoring distributed systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [RED method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals)

