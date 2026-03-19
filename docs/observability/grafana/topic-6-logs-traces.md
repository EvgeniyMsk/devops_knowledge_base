# 6. Логи и трассы в Grafana (Loki, Tempo)

---
**Метрик** недостаточно, чтобы ответить «что именно пошло не так в запросе X». **Grafana** объединяет **Prometheus**, **Loki** и **Tempo** (или другой backend трейсов) в одном **Explore** и на дашбордах. Ниже — идея **Loki**, **Tempo**, связка **логи ↔ метрики ↔ трассы** и практика: datasource, **LogQL** по лейблам, переход **метрика → лог → trace**.

---

## Зачем выходить за пределы метрик

| Сигнал | Вопрос |
|--------|--------|
| **Метрики** | Сколько ошибок? Растёт ли latency? Нарушен ли SLO? |
| **Логи** | Какой stack trace, какой `user_id`, какое сообщение в момент инцидента? |
| **Трассы** | Какой **span** медленный, где сеть, какие downstream-вызовы? |

**Best practice:** корреляция по **`trace_id`** (и при необходимости **`span_id`**) в логах и метриках (**exemplars**) — стандартный путь в современных стеках.

---

## Grafana Loki: кратко

- Хранилище логов с **индексом по лейблам** (как Prometheus — по наборам labels), а не полнотекст по всему объёму как в классическом ELK на каждую строку.
- Сбор: **Promtail**, **Fluent Bit**, **OpenTelemetry Collector** → push в Loki.
- Запросы: **LogQL** в **Explore** или панели типа **Logs**.

**Production:** не превращать высококардинальные поля (`request_id` на каждую строку) в **лейблы** — взрыв индекса и стоимость; часть полей оставлять в **JSON body** и парсить при необходимости.

---

## Tempo (или совместимый backend)

**Grafana Tempo** — хранилище **трасс** по **trace id**; приём через **OTLP**, **Jaeger**, **Zipkin**. В Grafana добавляется datasource **Tempo**, в **Explore** строится **trace** waterfall.

**Best practice:** семплирование трейсов в **collector** (например 1–5% при высоком RPS) + **всегда** сохранять **ошибочные** трассы, если политика позволяет.

---

## Связка: логи ↔ метрики ↔ трассы

| Шаг | Что настраивают |
|-----|------------------|
| **Метрика → лог** | **Data links** на панели Prometheus: ссылка в Explore Loki с подстановкой `pod`, `namespace`, окна времени |
| **Лог → trace** | В **Loki** парсинг **`trace_id`** из JSON (**derived fields**) или шаблон ссылки на Tempo datasource |
| **Метрика → trace** | **Exemplars** в Prometheus (гистограммы) + datasource Tempo в Grafana |

---

## Практика: подключить Loki в Grafana

Провижининг datasource (фрагмент):

```yaml
apiVersion: 1
datasources:
  - name: Loki
    uid: loki-main
    type: loki
    access: proxy
    url: http://loki-gateway.monitoring.svc:3100
    jsonData:
      maxLines: 1000
    editable: false
```

URL зависит от способа развёртывания (**single binary**, **read/write path**, **helm**).

---

## Практика: поиск логов по лейблам (LogQL)

Сначала фильтр **stream** по лейблам (как селектор метрик), затем **pipeline** по содержимому:

```logql
# Все логи пода api в namespace prod
{namespace="prod", app="api"}

# Только строки со словом ERROR (регистр настраивается)
{namespace="prod", container="api"} |= "ERROR"

# Исключение шума
{job="varlogs"} != "healthcheck"
```

**Комментарий:** `{job="..."}` — типичный набор лейблов с **Promtail**; ваши имена могут отличаться (`cluster`, `stream`).

Для **агрегата** по потоку (сколько строк за интервал совпало с фильтром):

```logql
sum(count_over_time({app="api"} |= "ERROR" [1m]))
```

**Комментарий:** для «ошибок в секунду» оформляйте **recording rule** в Loki или см. актуальный **LogQL** `rate`/`bytes_rate` в документации вашей версии — синтаксис метрик из логов эволюционировал.

---

## Практика: переход метрика → лог → trace

1. На панели **HTTP 5xx** добавьте **data link** «View logs» → URL Explore:
   - datasource Loki;
   - запрос с переменными `${__field.labels.namespace}`, `${__field.labels.pod}`.
2. В **Loki** включите **derived field** «traceID»: regexp из JSON `trace_id":"([a-f0-9]+)"` → ссылка на **Tempo** по этому id (настройка в datasource Loki → **Derived fields**).
3. В **Prometheus** при поддержке приложения настройте **exemplars** для гистограммы latency и привяжите **Tempo** datasource в панели — клик по точке откроет **trace**.

**Production:** единый формат **`trace_id`** в логах (W3C **traceparent** / OpenTelemetry) на всех сервисах; иначе drill-down ломается на границе команд.

---

## Чеклист

| # | Практика |
|---|----------|
| 1 | Лейблы Loki **низкой** кардинальности; высокая — в parsed json |
| 2 | Retention и квоты на Loki/Tempo согласованы с **cost** и **compliance** |
| 3 | RBAC: кто видит **prod** логи в Grafana |
| 4 | Проверить корреляцию на **одном** заранее известном `trace_id` в staging |
| 5 | Алерты по логам — аккуратно (шум); часто **метрика** + drill-down в лог |

---

## Дополнительные материалы

- [Loki](https://grafana.com/docs/loki/latest/)
- [LogQL](https://grafana.com/docs/loki/latest/logql/)
- [Tempo](https://grafana.com/docs/tempo/latest/)
- [Correlating traces with logs](https://grafana.com/docs/grafana/latest/explore/correlations/)
- Обзор в этом проекте: [Loki](../loki/loki.md)
