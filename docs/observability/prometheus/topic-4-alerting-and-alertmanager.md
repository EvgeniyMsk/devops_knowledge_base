# 4. Алертинг (Alertmanager)

---
Для **production** недостаточно графиков: нужны **правила** в Prometheus, понятный **жизненный цикл** алерта и **Alertmanager** с маршрутизацией, группировкой, **подавлением** шума (**inhibition**) и контролируемым **silence**. Ниже — суть компонентов, примеры **alert/recording** rules и фрагменты **маршрутизации** в Slack / Telegram / email.

---

## Alert rules (Prometheus)

Правила хранят в файлах (часто `alerts.yml`) и подключают через `rule_files` в `prometheus.yml`.

| Поле | Назначение |
|------|------------|
| **`expr`** | Условие в **PromQL**; при срабатывании создаётся **активная** серия алерта |
| **`for`** | Сколько условие должно держаться, прежде чем перейти в **Firing** (антифлаппинг) |
| **`labels`** | Фиксированные лейблы (`severity`, `team`…) для маршрутизации |
| **`annotations`** | Человекочитаемый текст, ссылки на **runbook** |

```yaml
groups:
  - name: node.rules
    interval: 30s
    rules:
      - alert: HighCPU
        expr: |
          100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Высокая загрузка CPU на {{ $labels.instance }}"
          description: "Idle < 20% в течение 5m. Runbook https://wiki.example/runbooks/high-cpu"
```

**Pod restart** (метрики **kube-state-metrics** / типовой `kube_pod_container_status_restarts_total` — имя и лейблы сверьте с вашим стеком):

```yaml
      - alert: PodRestarting
        expr: increase(kube_pod_container_status_restarts_total[15m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Рестарт контейнера {{ $labels.pod }} в ns {{ $labels.namespace }}"
```

**Всплеск HTTP 5xx** (метрика `http_requests_total` должна содержать лейбл `status`):

```yaml
      - alert: HTTP5xxSpike
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (job) (rate(http_requests_total[5m])) > 0.05
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Доля 5xx > 5% по job {{ $labels.job }}"
```

Best practices:

- **`for`** подбирать под SLO и шум (часто 2–15 минут для инфраструктуры);
- в **annotations** — **runbook**, не пароли;
- одна **серьёзность** (`severity`) — единая семантика во всех правилах.

Подключение в `prometheus.yml`:

```yaml
rule_files:
  - /etc/prometheus/alerts/*.yml
```

---

## Recording rules

**Recording rules** заранее считают тяжёлый PromQL и сохраняют результат как **новую метрику** — быстрее дашборды и проще алерты.

```yaml
groups:
  - name: agg.http
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

Комментарий: имя записи обычно следует шаблону `level:metric:operations` (рекомендация Prometheus).

---

## Жизненный цикл алерта

| Состояние | Смысл |
|-----------|--------|
| **Inactive** | Условие `expr` ложно |
| **Pending** | Условие истинно, но не выдержан интервал **`for`** |
| **Firing** | Отправка в **Alertmanager** (если настроено `alerting`) |

После восстановления метрики Alertmanager получает **resolved**; на стороне каналов уведомлений настройте, показывать ли «зелёные» сообщения.

---

## Alertmanager: маршрутизация, группировка, inhibition, silence

### Маршрутизация (`route`) и получатели (`receivers`)

Идея: по **лейблам** (`severity`, `team`, `cluster`) выбрать **Slack / email / Telegram** и т.д.

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: default
  group_by: [alertname, cluster]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers:
        - severity="critical"
      receiver: pager-slack
      continue: false
    - matchers:
        - team="payments"
      receiver: payments-email

receivers:
  - name: default
    slack_configs:
      - channel: "#alerts-infra"
        send_resolved: true
        title: "{{ .CommonLabels.alertname }}"
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"

  - name: pager-slack
    slack_configs:
      - channel: "#pager"
        username: Alertmanager

  - name: payments-email
    email_configs:
      - to: "payments-oncall@example.com"
        send_resolved: false
```

Комментарий: для **Slack** часто используют **Incoming Webhook** через `api_url` в секрете; не храните URL в Git.

### Telegram (поддерживается в современных версиях Alertmanager)

```yaml
  - name: telegram
    telegram_configs:
      - bot_token_file: /etc/alertmanager/secrets/telegram_bot_token
        chat_id: -1001234567890
        message: "{{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}"
```

Best practice: `bot_token` только из **файла** или переменной окружения, не в репозитории.

### Grouping (группировка)

- **`group_by`** — слияние похожих алертов в **одно** уведомление;
- **`group_wait`** — пауза перед первой отправкой (есть время собрать «пачку»);
- **`group_interval`**, **`repeat_interval`** — как часто повторять напоминания.

### Inhibition (подавление)

Скрывать «дочерние» алерты при активном «родителе» (например нода down → не спамить по каждому Pod).

```yaml
inhibit_rules:
  - source_matchers:
      - alertname="NodeExporterDown"
    target_matchers:
      - severity="warning"
    equal: ["instance"]
```

### Silencing (молчание)

- UI Alertmanager → **Silences**;
- CLI **`amtool silence add`**;
- на время работ **окно**, с матчерами по лейблам.

Best practice: обязательные поля **author**, **comment**, **тикет** change-management; срок окончания silence.

---

## Проверка конфигурации

```bash
promtool check rules /etc/prometheus/alerts/prod.yml
promtool check config /etc/prometheus/prometheus.yml

amtool check-config /etc/alertmanager/alertmanager.yml
```

В production конфигурации проходят **CI** (`promtool` / `amtool`) перед выкладкой.

---

## Сводный production checklist

| # | Практика |
|---|----------|
| 1 | У **каждого** Firing-алерта есть runbook или ссылка |
| 2 | Единые **`severity`** и маршруты critical vs warning |
| 3 | `group_by` и тайминги согласованы с on-call (без дубля) |
| 4 | Inhibition для каскадных отказов |
| 5 | Silences только с тикетом и TTL |

---

## Дополнительные материалы

- [Alerting overview](https://prometheus.io/docs/alerting/latest/overview/)
- [Configuration (Alertmanager)](https://prometheus.io/docs/alerting/latest/configuration/)
- [Notification examples](https://prometheus.io/docs/alerting/latest/notifications/)

