# 1. База: Grafana и место в Observability-стеке

---
**Grafana** — платформа **визуализации** и единой «точки входа» для операторов: метрики, логи и трассы из разных **источников данных** (datasources), дашборды, алерты, папки и права. Цель этого материала — понять **роль Grafana** рядом с Prometheus/Loki/Jaeger, различие **metrics / logs / traces** и поднять **локальный** контур: Docker → Prometheus datasource → первый дашборд (**CPU**, **память**).

---

## Роль Grafana в мониторинге и observability

| Идея | Комментарий |
|------|-------------|
| **Визуализация** | Графики, таблицы, стат-панели, heatmap — поверх запросов к источникам |
| **Агрегация сигналов** | Один UI для Prometheus, Loki, Elasticsearch, InfluxDB и др. |
| **Операторский сценарий** | On-call открывает **дашборд** и **Explore**, а не сырой API каждого бэкенда |
| **Не замена хранилища** | Grafana **не** обязана долго хранить сырые метрики — это задача Prometheus / Mimir / VictoriaMetrics |

**Production best practice:** Grafana в **production** держите за **TLS**, с **GitOps/провижинингом** дашбордов и datasource’ов, **RBAC** и отдельными организациями/папками для команд; секреты к БД — в **Vault**/K8s Secret, не в Git в открытом виде.

---

## Metrics vs Logs vs Traces

| Сигнал | Вопрос | Типичные инструменты | В Grafana |
|--------|--------|----------------------|-----------|
| **Metrics** | Сколько? Как быстро растёт? Контролируем ли бюджет ошибок? | Prometheus, InfluxDB, Graphite | Time series панели, **PromQL**, alerting |
| **Logs** | Что произошло в момент времени? Текст ошибки? | **Loki**, Elasticsearch, Splunk | **LogQL**, панели логов, корреляция с метриками |
| **Traces** | Какой путь запроса и где задержка? | **Jaeger**, Tempo, Zipkin | **TraceQL**/ссылки на drill-down в UI трейсинга |

Связка **три столпа**: метрики дают **симптомы и SLO**, логи — **контекст** инцидента, трейсы — **узкое место** в распределённом вызове.

---

## Основные источники данных (datasources)

| Datasource | Когда выбирают | Production-замечание |
|------------|----------------|----------------------|
| **Prometheus** | Метрики pull-модели, Kubernetes, exporters | Частый **default**; согласовать `scrape_interval` и **`timeInterval`** в Grafana |
| **Loki** | Лёгкие логи с **лейблами** (часто с Promtail) | Не дублировать полнотекст без нужды; индекс по лейблам |
| **Elasticsearch** | Полнотекст, старые стеки, Kibana-соседство | Нагрузка и стоимость кластера ES; политика retention |
| **InfluxDB** | Высокая кардинальность, IoT, legacy | Следить за схемой bucket’ов и cost |

В корпоративных контурах часто добавляют **Tempo**, **ClickHouse**, **MySQL** (бизнес-отчёты) — принцип тот же: **один** Grafana, **много** datasource UID’ов.

---

## Практика: Docker — Grafana + Prometheus + node_exporter

Минимальный **docker-compose** для упражнения (не для публичного интернета без доработки):

```yaml
# docker-compose.yml (фрагмент)
services:
  prometheus:
    image: prom/prometheus:v2.52.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"

  node_exporter:
    image: prom/node-exporter:v1.8.1
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"

  grafana:
    image: grafana/grafana:11.0.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: "Смените_в_лабе_и_никогда_так_в_проде"
    depends_on:
      - prometheus
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ["node_exporter:9100"]
```

**Запуск:** `docker compose up -d`, Prometheus UI `http://localhost:9090/targets` — цель **UP**.

---

## Подключить Prometheus в Grafana

1. Откройте `http://localhost:3000`, войдите как admin.
2. **Connections → Data sources → Add new → Prometheus**.
3. **URL:** `http://prometheus:9090` (из контейнера Grafana — имя сервиса compose) или `http://host.docker.internal:9090` при разнесённом запуске.
4. **Save & test** — должно быть зелёное «Successfully queried».

**Production:** провижининг файлом (идемпотентно, GitOps):

```yaml
# provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    uid: prom-lab
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 15s
    editable: false
```

Смонтируйте каталог провижининга: `--volume ./grafana/provisioning:/etc/grafana/provisioning`.

---

## Первый дашборд: CPU и память (node_exporter)

Создайте **Dashboard → Add visualization**, datasource **Prometheus**.

**Загрузка CPU (упрощённо, все ядра):**

```promql
# Доля «не idle» CPU по инстансу (0–1 → умножьте на 100 в легенде или Unit)
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

Для панели: **Unit → Percent (0.0-1.0)** или выражение `* 100` и **Percent (0-100)**.

**Использование памяти:**

```promql
# Доля занятой памяти (MemTotal - MemAvailable оценка «доступно»)
1 - (
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

**Best practice:** в production добавляйте лейблы **`env`**, **`cluster`**, **`job`** и **переменные дашборда** (`$instance`, `$job`), иначе при двух серверах графики смешаются.

Импорт готовой пашели: **Dashboards → Import → ID 1860** (Node Exporter Full) — удобно сравнить со своими панелями.

---

## Краткий чеклист (вводный production)

| # | Практика |
|---|----------|
| 1 | Datasource и дашборды из **provisioned** конфигов; **не** править бой только кликами без бэкапа JSON |
| 2 | **RBAC**: кто может менять алерты и датасорсы |
| 3 | Версионирование дашбордов (**JSON** в Git); один источник правды |
| 4 | Отдельные Grafana для **prod** и **dev** или строгие папки/орги |
| 5 | Дальше — **Loki**/трейсы и корреляция; см. разделы Observability в документации |

---

## Дополнительные материалы

- [Grafana документация](https://grafana.com/docs/grafana/latest/)
- [Data sources](https://grafana.com/docs/grafana/latest/datasources/)
- [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- Связка с Prometheus: [визуализация в треке Prometheus](../prometheus/topic-6-grafana-visualization.md)
