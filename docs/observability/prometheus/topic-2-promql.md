# 2. PromQL

---
**PromQL** — язык запросов к метрикам Prometheus. Для уровня **Middle+** нужно уверенно различать **instant** и **range** векторы, применять **`rate` / `irate` / `increase`**, агрегировать через **`by` / `without`** и собирать типовые панели: **ошибки**, **насыщение**, **латентность по гистограммам**. Ниже — концентрат с комментариями и примерами «как в production».

---

## Типы данных

| Тип | Смысл | Пример |
|-----|--------|--------|
| **Instant vector** | Набор time series — **одно значение на момент** запроса (или «сейчас» в UI) | `http_requests_total` |
| **Range vector** | Для каждого ряда — **окно сэмплов** за интервал; используется **внутри** функций вроде `rate` | `http_requests_total[5m]` |
| **Scalar** | Одно число | `42` |

Best practice: **`range vector` нельзя** просто вывести в Graph — только передать в функцию, которая его принимает (`rate`, `increase`, `avg_over_time`…).

---

## Базовые функции: `rate()`, `irate()`, `increase()`

Применяются к **counter**-ам (монотонно растущим счётчикам).

| Функция | Когда использовать | Комментарий |
|---------|-------------------|-------------|
| **`rate()[range]`** | Средняя скорость роста **за окно** — **основной** выбор для алертов и дашбордов | Окно **≥ ~4× scrape_interval** (часто `5m`) |
| **`irate()[range]`** | Очень **короткий** интервал, «мгновенная» скорость по двум последним точкам | Для **графиков в реальном времени**; для алертов может быть шумным |
| **`increase()[range]`** | Прирост счётчика за окно (приближённо) | Удобно для «сколько запросов за час» в визуализации, не путать с точным бухучётом |

```promql
# Среднее число событий в секунду за 5 минут
rate(http_requests_total{job="api"}[5m])
```

```promql
# Прирост за час (для отчётности — помните про экстраполяцию на границах окна)
increase(http_requests_total{job="api"}[1h])
```

Production: для **стейл меток** и сбойных scrape `rate` уже учитывает сбои лучше «ручного» подсчёта; не используйте `rate` у **gauge**.

---

## Арифметика и агрегаторы: `sum`, `avg`, `max`, `min`

```promql
sum(rate(http_requests_total[5m]))

avg(node_cpu_seconds_total{mode="idle"})

max(min_over_time(up[5m]))
```

Комментарий: агрегаторы **снимают лейблы**, кроме указанных в `by ()` / `without ()`.

---

## Агрегация: `by` и `without`

| Модификатор | Смысл |
|-------------|--------|
| **`sum by (pod, namespace) (...)`** | Сумма, **сохранить** перечисленные лейблы |
| **`sum without (instance) (...)`** | Сумма, **убрать** `instance`, остальные лейблы сохраняются насколько возможно |

```promql
sum by (job, env) (rate(http_requests_total[5m]))
```

Best practice: явный **`by (...)`** читается лучше в больших запросах и снижает риск неожиданно «схлопнуть» важные лейблы.

---

## Фильтрация: селекторы лейблов

```promql
# Равенство
http_requests_total{job="api", method="GET"}

# Регулярное выражение (совпадение)
http_requests_total{status=~"5.."}

# Отрицание
http_requests_total{status!~"2.."}

# Исключить пустой job (частый паттерн)
container_cpu_usage_seconds_total{container!=""}
```

Production: не задавайте в правилах **слишком высококардинальные** фильтры (уникальный `request_id` в лейбле — антипаттерн на стороне приложения).

---

## Важные паттерны

### Error rate (доля ошибок)

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
/
sum(rate(http_requests_total[5m])) by (job)
```

Комментарий: при **нулевом** знаменателе в PromQL получится `NaN` — в алертах иногда добавляют `or vector(0)` или проверку через `unless`.

### Saturation (насыщение ресурса)

Идея: использование **относительно лимита** — CPU, память, диск, очередь.

```promql
# Утилизация памяти на ноде (node_exporter)
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

### Latency: histogram и квантиль

Гистограммы дают суффиксы `_bucket`, `_sum`, `_count`. Квантиль по **леке** (`le`):

```promql
histogram_quantile(
  0.95,
  sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
)
```

Best practice: квантили по **агрегированным** гистограммам — **статистическая оценка** (лучше смотреть по `job`/`route` разумной гранулярности); на скользящих окнах не забывайте про минимальную выборку.

---

## Практика: готовые запросы

### CPU usage по pod (Kubernetes / cAdvisor-стиль метрик)

Имена метрик зависят от версии `kubelet`/`cadvisor`; типовой вариант:

```promql
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="", pod!=""}[5m])
)
```

### Memory usage по ноде (node_exporter)

Нагрузка ОС в процентах (упрощённо, без учёта кэша как «свободной» памяти в некоторых моделях):

```promql
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

Абсолютное использование:

```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

### HTTP: доля 5xx

См. блок Error rate выше; вариант с процентами:

```promql
100 * (
  sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
  /
  sum(rate(http_requests_total[5m])) by (job)
)
```

### Requests per second (RPS)

```promql
sum(rate(http_requests_total[5m])) by (job)
```

Для разбивки по маршруту добавьте лейбл из вашей инструментации (`handler`, `route`):

```promql
sum(rate(http_requests_total[5m])) by (job, handler)
```

---

## Production checklist по PromQL

| # | Практика |
|---|----------|
| 1 | Для counter — **`rate`/`increase`**, не сырые значения |
| 2 | Окно `[5m]` согласовать с `scrape_interval` и SLO |
| 3 | Алерты на **burn rate** / error budget строить на тех же запросах, что и графики (или recording rules) |
| 4 | `histogram_quantile` только с **`rate(..._bucket[...])`** |
| 5 | Документировать «откуда взялись лейблы» в дашборде для он-call |

---

## Дополнительные материалы

- [Querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Operators](https://prometheus.io/docs/prometheus/latest/querying/operators/)
- [Functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)
</think>


<｜tool▁calls▁begin｜><｜tool▁call▁begin｜>
StrReplace