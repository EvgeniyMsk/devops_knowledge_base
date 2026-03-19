# 5. Мониторинг и алертинг

---
В **production** недостаточно «кластер поднят»: нужны **метрики**, **дашборды** и **алерты** по задержке репликации, доступности **primary**, нагрузке (**opcounters**), соединениям, кэшу WiredTiger и диску. Типичный стек — **Prometheus** + **Grafana** + **MongoDB Exporter**. Ниже — какие сигналы смотреть и минимальные примеры конфигурации.

---

## Стек: Prometheus, Grafana, MongoDB Exporter

| Компонент | Роль |
|-----------|------|
| **MongoDB Exporter** | Собирает метрики с `mongod`/`mongos` в формате **Prometheus** |
| **Prometheus** | Хранение временных рядов, правила **recording/alerting** |
| **Grafana** | Дашборды, визуализация, можно Alerting к тому же Prometheus |

Популярные реализации экспортера: **[Percona MongoDB Exporter](https://github.com/percona/mongodb_exporter)** (поддержка современных версий MongoDB). В managed-сервисах (Atlas и др.) часть метрик уже в облачной консоли, но **единый** Prometheus может всё равно быть нужен для корпоративного мониторинга.

**Best practice:** отдельный **технический пользователь** БД с правами только на **метрики** (`clusterMonitor` / минимально достаточные роли), не root; URI хранить в **секретах**.

---

## Запуск экспортера (идея)

Через **Docker** (параметры см. README выбранного экспортера):

```bash
# Пример: Percona exporter, совместимая схема URI с replica set
docker run -d --name mongodb-exporter \
  -p 9216:9216 \
  percona/mongodb_exporter:0.40 \
  --mongodb.uri='mongodb://monitor:REDACTED@mongo1:27017,mongo2:27017,mongo3:27017/admin?replicaSet=rs0'
```

Флаги **`--collect-all`** или выборочные коллекторы включайте осознанно — избыток метрик удорожает **cardinality** в Prometheus.

### Скрапинг в Prometheus

```yaml
# prometheus.yml (фрагмент)
scrape_configs:
  - job_name: mongodb
    scrape_interval: 30s
    static_configs:
      - targets:
          - mongodb-exporter.prod:9216
        labels:
          cluster: shop-rs0
          env: production
```

**Production:** для Kubernetes — **ServiceMonitor** (Prometheus Operator) вместо статических target’ов.

---

## Ключевые метрики

### Opcounters (операции в секунду)

Показывают профиль нагрузки: **insert**, **query**, **update**, **delete**, **getmore**. Резкие **аномалии** или ноль при ожидаемой нагрузке — повод проверить приложение, сеть или **primary**.

Имена в Prometheus зависят от экспортера (часто с префиксом `mongodb_ss_opcounters_*` или аналогом для `serverStatus`).

### Connections

- Текущее число соединений vs **`maxIncomingConnections`**.
- **Best practice:** пулы в приложениях с **верхней границей**; алерт при доле используемых соединений к лимиту (например >80% уверенного порога).

### Replication lag

- Задержка вторичных относительно **oplog** primary — критична для чтений с `secondary` и для **RPO**.
- Метрики вида **heartbeat lag** / **replset_member replication lag** (имена у экспортера уточняйте в документации и в **`/metrics`**).

**Production:** пороги зависят от SLA; для строгой согласованности чтений с secondary лаг должен быть **малым и стабильным**.

### Cache usage (WiredTiger)

- Использование **cache** движком, **dirty** / **pressure** — признаки нехватки RAM или тяжёлых запросов без индексов.
- Смотрите также **page faults** и **eviction** (если экспортированы).

### Disk I/O

- **Задержки**, **utilization** диска, **свободное место** — вне Prometheus часть снимается **node exporter** / облачными агентами.
- MongoDB: рост **data** каталога, **компактация** не «лечит» всё; планирование ёмкости заранее.

---

## Практика: дашборд в Grafana

1. Импортируйте готовый дашборд по ID из **grafana.com** для вашего экспортера (например поиск `MongoDB Percona`).
2. Минимальные **панели** для production-обзора:
   - статус членов RS (кто **PRIMARY**/SECONDARY);
   - **replication lag** по членам;
   - **opcounters**, **connections**;
   - **WiredTiger cache** (или аналог из метрик);
   - **disk usage** с хоста (через node_exporter или облако).

**Best practice:** один дашборд на **кластер** с переменной `cluster` / `job`, чтобы не плодить копии.

---

## Алерты: replication lag и недоступность primary

Пример **Prometheus Alerting rules** (имена метрик **замените** на фактические из вашего экспортера после `curl exporter:9216/metrics`):

```yaml
groups:
  - name: mongodb
    rules:
      # Репликационный лаг (пример: метрика в секундах; порог под ваш SLA)
      - alert: MongoDBReplicationLagHigh
        expr: mongodb_rs_members_optime_lag > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Высокий replication lag на MongoDB"
          description: "Член RS отстаёт более чем на {{ $value }}с. Проверить сеть, нагрузку и диск вторичных."

      # Нет PRIMARY в job/cluster (пример через счётчик членов в состоянии PRIMARY)
      # Замените metric/label на реальные из exporter
      - alert: MongoDBNoPrimary
        expr: sum by (cluster) (mongodb_rs_members_state{member_state="PRIMARY"}) == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB: нет PRIMARY в кластере"
          description: "Нет доступного PRIMARY для {{ $labels.cluster }}. Риск простоя записи."
```

**Важно:** в разных версиях экспортера метрики называются по-разному; **первый шаг** — зафиксировать **реальные имена** и **лейблы**, затем подстроить `expr`.

Дополнительно полезны алерты:

- мало **свободного места** на томе данных;
- **connections** близко к лимиту;
- частые **elections** (если метрика доступна).

---

## Production checklist (мониторинг)

| # | Практика |
|---|----------|
| 1 | Единый источник правды: SLO по **lag**, **uptime** записи, диску |
| 2 | Алерты с **runbook** (что проверить: `rs.status()`, логи, сеть) |
| 3 | Не хранить пароли экспортера в репозитории; ротация учёток |
| 4 | Тест «алерт сработал» в staging; избегать **alert fatigue** |
| 5 | Корреляция с **логами** и **трейсами** приложения при деградации |

---

## Дополнительные материалы

- [Percona MongoDB Exporter](https://github.com/percona/mongodb_exporter)
- [MongoDB — serverStatus](https://www.mongodb.com/docs/manual/reference/command/serverStatus/)
- [MongoDB — replSetGetStatus](https://www.mongodb.com/docs/manual/reference/command/replSetGetStatus/)
- [Prometheus alerting](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
