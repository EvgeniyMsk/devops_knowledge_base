# 6. Grafana (визуализация)

---
**Prometheus** хранит метрики и считает алерты, но **без дашбордов** сложно держать в голове картину системы. **Grafana** — стандартная оболочка: **datasource** на Prometheus, **дашборды** с PromQL, **переменные** для окружений и узлов, при необходимости — **алертинг**. Ниже — как связать компоненты, чем отличаются алерты Grafana и **Alertmanager**, и как собрать панели для **ноды**, **приложения** и **кластера Kubernetes**.

---

## Datasource Prometheus

| Параметр | Production-комментарий |
|----------|-------------------------|
| **URL** | Внутрикластерный `http://prometheus-operated:9090` или сервис с mTLS/NetworkPolicy |
| **Access** | **Server** (Grafana backend ходит в Prometheus) чаще, чем Browser |
| **Auth** | Basic/mTLS при внешнем Prometheus; в K8s — сервисный аккаунт / sidecar |
| **`timeInterval`** | Согласовать с **`scrape_interval`** (например `30s`), чтобы UI не «дёргал» тяжёлее реальности |

### Провижининг datasource (файл)

```yaml
# provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    uid: prom-main
    type: prometheus
    access: proxy
    url: http://kube-prom-kube-prometheus-prometheus.monitoring.svc:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      timeInterval: 30s
    editable: false
```

Комментарий: `url` и имя сервиса зависят от релиза **Helm**; зафиксируйте **UID** datasource — он нужен в JSON дашбордов при экспорте.

---

## Dashboards: структура

- **Row** — логическая группа панелей (Node / App / K8s).
- **Panel** — тип **Time series**, **Stat**, **Gauge**, **Table**; запросы на **PromQL**.
- **Repeat** панели по переменной (например по всем `instance`).

Best practices:

- дашборды хранить в **Git** (json или as-code **Grizzly/Terraform**);
- общие панели — **библиотечные** panel library (Grafana 10+);
- не дублировать один и тот же тяжёлый запрос в 20 виджетах — вынести в **recording rule**.

---

## Variables (переменные)

| Тип | Применение |
|-----|------------|
| **Query** | Список `namespace`, `pod`, `instance` из `label_values(...)` |
| **Custom** | Фиксированный список сред (`prod`, `stage`) |
| **Interval** | Динамический `$__interval` для `rate()` в зависимости от масштаба графика |

Примеры **Query variable**:

```promql
label_values(node_uname_info, nodename)
```

```promql
label_values(kube_namespace_labels, namespace)
```

В панелях используйте `$instance`, `$namespace` в PromQL:

```promql
100 - (
  avg by (instance) (rate(node_cpu_seconds_total{instance=~"$instance",mode="idle"}[$__rate_interval])) * 100
)
```

Комментарий: **`$__rate_interval`** удобен во **встроенных** дашбордах Grafana для согласования с zoom; иначе фиксируйте `[5m]` осознанно.

---

## Алертинг: Grafana vs Prometheus

| Аспект | **Prometheus + Alertmanager** | **Grafana alerting** |
|--------|-------------------------------|-------------------------|
| Источник правил | `prometheus.yml` / recording+alert rules | UI или provisioning |
| Маршрутизация | **Alertmanager** (inhibit, silence) | Contact points, mute timings |
| Мульти-дatasource | Нет (только метрики Prom) | Да (Loki, CloudWatch…) |
| Типичный выбор в prod | **Инфраструктурные** SLO, единый on-call поток | **Продуктовые** панели, составные условия |

Best practice: не создавать **две копии** одного и того же критичного условия без согласования — иначе двойные звонки. Часто: **алерты данных** в Prometheus, **визуализация запасов** и **аннотации** в Grafana.

---

## Практика: собрать дашборд

### Блок «Node metrics»

| Панель | Идея PromQL |
|--------|----------------|
| CPU | `100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[$__rate_interval])) * 100` |
| RAM | `100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` |
| Диск | `100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)` |

### Блок «App metrics»

| Панель | Идея PromQL |
|--------|----------------|
| RPS | `sum by (job) (rate(http_requests_total[$__rate_interval]))` |
| Ошибки 5xx | `sum(rate(http_requests_total{status=~"5.."}[$__rate_interval])) / sum(rate(http_requests_total[$__rate_interval]))` |
| Latency p95 | `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[$__rate_interval])))` |

### Блок «Kubernetes cluster» (kube-state-metrics / cAdvisor)

| Панель | Идея PromQL |
|--------|----------------|
| Pods not Ready | `sum by (namespace) (kube_pod_status_phase{phase!~"Running|Succeeded"})` |
| CPU по namespace | `sum by (namespace) (rate(container_cpu_usage_seconds_total{container!=""}[$__rate_interval]))` |
| Рестарты | `increase(kube_pod_container_status_restarts_total[$__rate_interval])` |

Имена метрик зависят от версии **kube-state-metrics** и **cadvisor** — подставьте свои после `Metrics explorer` в Prometheus.

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | **SSO** (OIDC/LDAP) и роли **Viewer/Editor** по командам |
| 2 | Запрет **anon** доступа в prod |
| 3 | Бэкап **SQLite**/БД Grafana или только Git-as-source для дашбордов |
| 4 | Тяжёлые запросы → **recording rules** |
| 5 | Переменные без «All» на огромных кластерах без лимита — риск таймаутов |

---

## Дополнительные материалы

- [Grafana — Prometheus datasource](https://grafana.com/docs/grafana/latest/datasources/prometheus/)
- [Dashboards: variables](https://grafana.com/docs/grafana/latest/dashboards/variables/)
- Отдельный обзор в курсе: [Grafana](../grafana/grafana.md) (если раздел заполнен)

