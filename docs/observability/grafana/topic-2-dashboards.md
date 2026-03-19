# 2. Дашборды и визуализация

---
Хороший дашборд в **production** отвечает на **операционный вопрос** за секунды («здоров ли кластер?», «где горит этот сервис?»), а не только выглядит «красиво». Ниже — типы **панелей**, **переменные (templating)**, **трансформации**, **drill-down** между экранами и практика: **Node Exporter**, обзор **Kubernetes** с фильтром по **namespace** / **pod**.

---

## Типы панелей

| Тип | Когда использовать | Production-замечание |
|-----|-------------------|----------------------|
| **Time series** | Тренды во времени (CPU, RPS, latency p95) | Задайте **min step** / не сглаживайте «в ноль» то, что нужно видеть при инциденте |
| **Stat / Gauge** | Одно число сейчас: ошибки/мин, свободное место | Для SLO часто **Stat** + пороговая раскраска; не смешивайте несопоставимые единицы |
| **Table** | Топ N по лейблам, список подов, сравнение версий | **Transformations** для сортировки/фильтра; тяжёлые запросы — с лимитом и алертом |

**Best practice:** на одном экране **3–5 ключевых** рядов для «обзора»; детали вынесите на **дочерние** дашборды или **Explore**, чтобы не перегружать коллег при онколле.

---

## Переменные (templating)

Переменные превращают статичный JSON в **переиспользуемый** дашборд: `$cluster`, `$namespace`, `$pod`.

### Источник значений из Prometheus

| Переменная | Тип запроса (пример) |
|------------|----------------------|
| **namespace** | `label_values(kube_pod_info, namespace)` или `metrics` с лейблом `namespace` |
| **pod** | `label_values(kube_pod_status_phase{namespace="$namespace"}, pod)` |

В UI: **Dashboard settings → Variables → Add variable**:

- **Type:** Query  
- **Data source:** Prometheus  
- **Query:** как в таблице выше  
- **Multi-value** / **Include All** — включайте осознанно (промежуток «All» может раздувать кардинальность в PromQL).

Пример **query** для списка нод из `node_exporter`:

```promql
label_values(node_uname_info, nodename)
```

### Зависимые переменные

Порядок важен: сначала **`namespace`**, затем **`pod`**, где вариант `pod` фильтрует по выбранному namespace (см. вторую строку таблицы).

**Production:** для больших кластеров включайте **regex** на переменной или **кастомный** список разрешённых namespace’ов, чтобы UI не подтягивал тысячи значений.

---

## Transformations

Преобразуют **сырой** ответ запроса до таблицы/графика без дублирования в PromQL:

- **Filter fields by name** — оставить только нужные колонки;
- **Sort by** — топ медленных эндпоинтов;
- **Merge** / **Join by field** — склеить два запроса по общему лейблу;
- **Add field from calculation** — процент от базовой колонки.

**Best practice:** если трансформация «магическая» для следующего инженера — добавьте **описание панели** (Description) или вынесите логику в **recording rule** в Prometheus.

---

## Drill-down (ссылки между дашбордами)

- **Data links** на панели: клик по серии → URL с подстановкой `${__field.labels.pod}`.
- **Dashboard links** в шапке: «Pod detail» с передачей `var-pod=...` в query string.

Пример шаблона ссылки (идея):

```text
/d/uuid-pod-detail/kubernetes-pod?var-namespace=${__field.labels.namespace}&var-pod=${__field.labels.pod}
```

В **production** используйте **стабильные UID** дашбордов (задаются при экспорте), чтобы ссылки не ломались при переименовании заголовка.

---

## Практика: обзор ноды (Node Exporter)

1. Импортируйте **Node Exporter Full** (ID **1860**) или соберите панели вручную.
2. Добавьте переменную **`instance`**:

```promql
label_values(node_cpu_seconds_total, instance)
```

3. В запросах используйте фильтр:

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance=~"$instance"}[5m])) * 100)
```

---

## Практика: обзор Kubernetes и фильтр namespace / pod

Типичный набор переменных для `kube-state-metrics` / cAdvisor (имена метрик зависят от экспортера):

| Имя | Query (ориентир) |
|-----|-------------------|
| `namespace` | `label_values(kube_namespace_labels, namespace)` |
| `pod` | `label_values(kube_pod_info{namespace="$namespace"}, pod)` |

Панель **CPU по поду** (пример для cAdvisor-стиля метрик):

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="$namespace",
    pod=~"$pod",
    container!="",
    image!=""
  }[5m])
)
```

**Multi-value** для `pod` полезен для сравнения нескольких подов; для инцидента часто достаточно **single** + быстрый переход по **drill-down**.

---

## Антипаттерны (кратко)

| Плохо | Лучше |
|-------|--------|
| 20 панелей без группировки и заголовков | Разбить на **rows** «Golden signals», «Capacity», «Errors» |
| Один огромный time range по умолчанию | Разумный **default** (6h) + быстрый «last 15m» |
| Переменная «All» на высококардинальных метриках | Ограничить regex, topk в запросе, отдельный «overview» |
| Копипаста дашбордов вместо **переменных** | Один шаблон + JSON в Git |

---

## Чеклист production-дашборда

| # | Практика |
|---|----------|
| 1 | **UID**, теги, папка, владелец команды |
| 2 | Переменные согласованы с **ролями** (кто видит какие namespace) |
| 3 | Описания панелей для нетривиальных PromQL |
| 4 | Ссылки на **runbook** и смежные дашборды |
| 5 | Версия в **Git** (или Grafana OSS + API) при изменении |

---

## Дополнительные материалы

- [Panels overview](https://grafana.com/docs/grafana/latest/panels-visualizations/)
- [Variables](https://grafana.com/docs/grafana/latest/dashboards/variables/)
- [Transformations](https://grafana.com/docs/grafana/latest/panels/transformations/)
- [Data links](https://grafana.com/docs/grafana/latest/panels-visualizations/configure-data-links/)
