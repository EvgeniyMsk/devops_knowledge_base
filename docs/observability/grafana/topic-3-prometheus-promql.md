# 3. Глубокая работа с Prometheus в Grafana

---
**Grafana** рисует то, что отдаёт **Prometheus**: без уверенного **PromQL** сложно строить панели **RPS**, **латентности (p95/p99)**, **доли ошибок** и дашборды **SLI/SLO**. Ниже — ключевые функции (**`rate`**, **`irate`**, агрегации **`sum by` / `avg by`**, **`histogram_quantile`**), проблемы **кардинальности**, **recording rules** и практические запросы для production-подобных графиков.

Детальная теория PromQL — в треке **Prometheus** → [PromQL](../prometheus/topic-2-promql.md); здесь — фокус на **связке запрос → панель Grafana**.

---

## `rate` и `irate`

Оба усредняют **скорость роста counter’а** за окно; в Grafana окно почти всегда задаётся в квадратных скобках в запросе.

| Функция | Поведение | Когда в Grafana |
|---------|-----------|-----------------|
| **`rate()[5m]`** | Средняя производная за окно; устойчивее к «ступенькам» | **RPS**, ошибки/сек, байты/сек — **дефолтный** выбор для панелей |
| **`irate()[5m]`** | Мгновенная скорость по двум последним точкам в окне | Реже; для **резких** всплесков и отладки, на длинном range может «шуметь» |

```promql
# HTTP RPS по path (пример имён метрик — подставьте свои из /metrics)
sum by (path) (rate(http_requests_total[5m]))

# Ошибки 5xx в секунду
sum(rate(http_requests_total{status=~"5.."}[5m]))
```

**Best practice:** окно **`[5m]`** согласуйте с **`scrape_interval`** (обычно ≥ **4× интервал сбора**). Слишком короткое окно → пропуски и «ломаная» линия.

---

## `sum by`, `avg by`, `max by`

**`by (...)`** оставляет указанные лейблы после агрегации — основной способ «свернуть» инстансы до сервиса или региона.

```promql
# Средняя утилизация CPU по namespace (пример kube-метрик)
sum by (namespace) (
  rate(container_cpu_usage_seconds_total{container!=""}[5m])
)

# Средняя латентность без потери инстанса (редко нужно «просто avg» — см. гистограммы ниже)
avg by (job) (process_resident_memory_bytes)
```

**Production:** избегайте **`sum(rate(...))`** без **`by`** на высококардинальных метриках для «общего» графика — одна линия «всё кластер» часто теряет диагностическую ценность; лучше переменная `$job` или **topk**.

---

## `histogram_quantile` и перцентили

**Histogram** в Prometheus хранится как **`_bucket`**, **`_sum`**, **`_count`**. Перцентиль **на гистограмме** считают по **bucket’ам** за окно:

```promql
# p95 латентности HTTP (имена le="" и метрик — как в вашем приложении)
histogram_quantile(
  0.95,
  sum by (le, path) (rate(http_request_duration_seconds_bucket[5m]))
)
```

```promql
# p99
histogram_quantile(
  0.99,
  sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
)
```

**Best practice:**

- Перцентили по **гистограмме** — аппроксимация; точные **квантили по сырым событиям** — в других системах или **native histogram** (если включена ваша цепочка).
- Не смешивайте на одной панели **разные** базовые метрики без легенды и единиц.

---

## Кардинальность: почему «ломается» Prometheus и Grafana

| Проблема | Следствие |
|----------|-----------|
| Лейбл **`user_id`** на каждом запросе | Взрыв числа временных рядов, рост памяти TSDB, медленные запросы в Grafana |
| Переменная **All** на метрике с сотнями `pod` | Тяжёлый запрос на каждый refresh дашборда |
| Высокий **scrape** + **низкий interval** в Grafana | Лишняя нагрузка на Prometheus |

**Mitigation:** согласование схемы метрик с **SRE**, **`limit`** / **`topk`**, **recording rules**, отказ от высококардинальных лейблов в **counter**’ах публичных API.

---

## Recording rules

Перенос «тяжёлого» PromQL на сторону Prometheus: заранее считать агрегаты с **меньшей** кардинальностью на запись — в Grafana запрос **короче и быстрее**.

```yaml
# prometheus.yml — фрагмент rule_files
groups:
  - name: recording_http
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_request_duration_seconds:p95
        expr: |
          histogram_quantile(
            0.95,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

В Grafana datasource остаётся Prometheus, но запрос к панели:

```promql
job:http_requests:rate5m
```

**Production:** recording rules в **Git**, одинаковый набор на **staging** и **prod**, алерт на **rule evaluation failures**.

---

## Практика: панели RPS, latency, error rate

### RPS (успешные запросы)

```promql
sum(rate(http_requests_total{status!~"5.."}[5m]))
```

### Error rate (доля 5xx от всех)

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

Умножьте на 100 в **Unit** или через `* 100` для **панели процента**.

### Latency p95 / p99

Используйте два запроса на одной time series панели с разными **`histogram_quantile`** (0.95 и 0.99) или две панели Stat для **SLO**.

---

## Дашборд SLI / SLO (идея структуры)

| Row / панель | Смысл |
|--------------|--------|
| **Доступность** | Доля успешных запросов за окно (SLI) vs целевой **SLO** (линия порога) |
| **Латентность** | p95/p99 vs бюджет (например p95 < 300 ms) |
| **Ошибки** | Error rate и топ `path`/`deployment` |
| **Бюджет ошибок** | При наличии recording rule или внешнего учёта — оставшийся budget за период |

**Best practice:** **SLO** фиксируйте в **документе/слотбуке**; Grafana показывает **факт** и нарушения, а не заменяет договорённость с бизнесом.

---

## Чеклист

| # | Практика |
|---|----------|
| 1 | Окна **`rate`** согласованы с scrape и с **Grafana min interval** |
| 2 | Перцентили — из **histogram_bucket**, не «сырой avg» по случайной метрике |
| 3 | Тяжёлые запросы → **recording rules** + простые панели |
| 4 | Ревью новых лейблов метрик на **кардинальность** |
| 5 | SLI/SLO дашборд связан с **алертами** и **runbook** |

---

## Дополнительные материалы

- [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Histograms and quantiles](https://prometheus.io/docs/practices/histograms/)
- [Recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- [Grafana — Query Prometheus](https://grafana.com/docs/grafana/latest/datasources/prometheus/#query-editor)
