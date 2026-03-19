# 1. База: архитектура и философия Prometheus

---
Цель — **быстро и основательно** понять, зачем нужен **Prometheus**, как устроены **временные ряды**, модель **pull** и когда нужен **Pushgateway**, как связаны **Prometheus Server**, **exporters** и **Alertmanager**. В конце — локальный подъём через **Docker Compose** с **node_exporter** и **cAdvisor**.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Time series (TS)** | Последовательность значений метрики во времени, идентифицируемая именем и **набором лейблов**. |
| **Metric** | Измеряемая величина (нагрузка CPU, задержка, счётчик запросов) в одном из **типов** (counter, gauge, histogram, summary). |
| **Label** | Пара ключ=значение, по которой фильтруют и группируют ряды (`instance`, `job`, `env`, `cluster`…). |
| **Scrape** | Периодический **HTTP GET** Prometheus к endpoint’у цели и разбор ответа (обычно **OpenMetrics** / text exposition). |
| **Target** | Конечная точка с метриками (exporter, ваш `/metrics`, service discovery). |
| **TSDB** | Встроенное **временное хранилище** Prometheus для сырых сэмплов и индекс по лейблам. |

---

## Что такое Prometheus и зачем он нужен

**Prometheus** — система мониторинга и алертинга с упором на **метрики** в формате временных рядов, **запросы** на языке **PromQL** и **self-hosted** хранение без обязательной внешней БД.

Production-смысл:

- **динамические** окружения (Kubernetes, autoscaling) — метки и service discovery;
- **операторский** сценарий: «что сломалось» + «какой SLO»;
- экосистема **exporters** и интеграция с **Grafana** и **Alertmanager**.

Best practice: Prometheus — не замена логам и трейсам; в связке **metrics + logs + traces** (см. остальные разделы Observability).

---

## Pull vs Push и Pushgateway

| Модель | Как работает | Когда уместна |
|--------|----------------|---------------|
| **Pull (основная)** | Prometheus сам ходит к `/metrics` по расписанию | Сервисы с HTTP, Kubernetes SD, стабильные сети |
| **Push** | Клиент отправляет данные в приёмник | Редкие **batch** job’ы, короткоживущие задачи без scrape window |

**Pushgateway** — компонент, куда **job** может **однократно запушить** метрики; Prometheus **забирает** их оттуда pull’ом.

Best practices для Pushgateway:

- **не** использовать как «общую точку» для всех сервисов — теряется семантика инстансов и растёт путаница;
- подходит для **CronJob** / CI шагов с явным `job` и лейблами, **очищать** устаревшие группы по runbook;
- в production ограничить доступ (сеть, auth, reverse proxy).

---

## Архитектура

```
┌──────────────────┐      scrape       ┌─────────────────┐
│ Targets:         │ ◄────────────────  │ Prometheus      │
│ exporters, apps  │      HTTP /metrics │ Server (TSDB +  │
│ Pushgateway      │                    │ PromQL engine)  │
└──────────────────┘                    └────────┬────────┘
                                                 │ alerts
                                                 ▼
                                        ┌─────────────────┐
                                        │ Alertmanager    │
                                        │ (dedup, route,  │
                                        │ notify)         │
                                        └─────────────────┘
```

| Компонент | Роль |
|----------|------|
| **Prometheus Server** | Scrape, хранение, правила записи/алертов, API и UI |
| **Exporters** | Переводят метрики ОС/ПО в формат Prometheus (`node_exporter`, `mysql_exporter`…) |
| **Alertmanager** | Группировка, подавление, маршрутизация в Slack/PagerDuty/email |

---

## Основные понятия: метрика, лейбл, scrape, target

- Одна **логическая метрика** + уникальный набор **лейблов** = одна **time series**.
- Высокая **кардинальность** лейблов (например `user_id` на каждого пользователя) **убивает** TSDB — в production так не делают.
- **`job`** и **`instance`** — стандартные лейблы: кто собирается и откуда.

Пример идеи PromQL (только иллюстрация запроса):

```promql
rate(http_requests_total{job="api"}[5m])
```

Комментарий: `rate()` применяют к **counter**-ам; интервал `[5m]` должен быть **≥ 4× scrape_interval** по общему правилу.

---

## Практика: локально через Docker Compose

Ниже — учебный стек: **Prometheus** + **node_exporter** + **cAdvisor**. Пути и версии образов обновляйте по политике безопасности в вашей организации.

### `docker-compose.yml`

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --web.enable-lifecycle
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prom_data:/prometheus
    depends_on:
      - node_exporter
      - cadvisor

  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    command:
      - --path.rootfs=/host
    pid: host
    volumes:
      - /:/host:ro,rslave
    ports:
      - "9100:9100"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    privileged: true
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prom_data: {}
```

Комментарий: **cAdvisor** на macOS с Docker Desktop часто даёт **урезанные** метрики хоста; для полноценной картины ОС используйте Linux VM или реальный сервер.

### `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["node_exporter:9100"]

  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
```

Запуск из каталога с файлами:

```bash
docker compose up -d
# UI Prometheus: http://localhost:9090  → Status → Targets (все должны быть UP)
```

---

## Production best practices (кратко)

| Практика | Зачем |
|----------|--------|
| Явный **`scrape_interval`** и ресурсы под TSDB | Стабильная задержка и предсказуемый диск |
| **Recording rules** для тяжёлых дашбордов | Меньше нагрузка в Grafana |
| **Relabeling** для нормализации `instance`, дропа ненужных лейблов | Контроль кардинальности |
| Отдельные **Alertmanager** и маршруты по severity | Меньше шума on-call |
| Ретенция и **remote write** (при необходимости) | Долгий архив во внешнее хранилище |

---

## Дополнительные материалы

- [Prometheus — документация](https://prometheus.io/docs/introduction/overview/)
- [Node exporter](https://github.com/prometheus/node_exporter)
- [cAdvisor](https://github.com/google/cadvisor)
- [Pushgateway](https://github.com/prometheus/pushgateway)

