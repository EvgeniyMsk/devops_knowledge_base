# 4. Алертинг в Grafana

---
На уровне **middle+** инженер должен уметь настроить алерты так, чтобы они **сигнализировали о поломках**, а не **засыпали шумом**. Ниже — **unified alerting** Grafana (современная модель), сравнение с правилами **Prometheus / Alertmanager**, **contact points**, **тишина (silence)** и практика: **CPU**, **недоступность сервиса**, **латентность**, **мультиусловие** и **recovery**.

См. также трек Prometheus: [Alertmanager](../prometheus/topic-4-alerting-and-alertmanager.md).

---

## Grafana Alerting (unified): суть

| Элемент | Назначение |
|---------|------------|
| **Contact point** | Куда слать уведомление: Slack, email, **PagerDuty**, webhook, Telegram и др. |
| **Notification policy** | Маршрутизация: по лейблам `severity`, `team`, **matching** дерева правил |
| **Alert rule** | Условие + интервал оценки + **For** (задержка до firing) |
| **Mute / Silence** | Временное отключение шума (окно обслуживания, ложные срабатывания) |

**Best practice:** один **источник правды** для критичных production-алертов либо **Alertmanager** (если уже стандарт команды), либо **Grafana** — смешение двух каналов на одни и те же симптомы даёт **дубли** и рассинхрон.

---

## Правила в Grafana vs Prometheus

| Критерий | **Grafana Alerting** | **Prometheus + Alertmanager** |
|----------|----------------------|-------------------------------|
| Где живёт правило | БД/объекты Grafana или **provisioning** | Файлы rules в Prometheus, маршруты — **Alertmanager** |
| Данные | Несколько **datasource** (Prometheus, Loki, …) в одном правиле | В основном **метрики** PromQL |
| **Silence** | В UI Grafana / API | **Alertmanager** silence |
| Корпоративный стандарт | Удобно, если всё визуально в одном месте | Типично для **GitOps** правил без UI |

**Production:** если правила уже в **Git** как `prometheus_rules.yml`, дублировать их «кликами» в Grafana обычно **не стоит** — лучше **визуализация** в Grafana, **firing** в Alertmanager или осознанный **mirroring** с документированием.

---

## Contact points (Slack, email, PagerDuty)

Настройка: **Alerting → Contact points → New**.

**Slack:** Incoming Webhook URL из workspace; в шаблоне сообщения используйте **`{{ .CommonLabels }}`**, **`{{ .StartsAt }}`** (синтаксис зависит от версии и типа интеграции — смотрите **Notification templates** в документации).

**Email:** SMTP в **Grafana config** или облачный сервис; для production — отдельный почтовый релей и лимиты.

**PagerDuty:** Integration Key как **secret** (K8s Secret / Vault), не в репозитории.

Пример фрагмента **Grafana config** для SMTP (идея):

```ini
[smtp]
enabled = true
host = smtp.example.com:587
user = grafana-alerts
password = ${GF_SMTP_PASSWORD}
from_address = grafana@example.com
```

---

## Silence и mute schedules

- **Silence** — разовое окно: «глушим `alertname=HighCPU` на `instance=node-1` до 02:00».
- **Mute timings** (в связке с **notification policies**) — регулярные окна, например плановые релизы.

**Best practice:** каждая тишина с **тикетом**/ссылкой на change; иначе реальный инцидент может «промолчать».

---

## Практика: высокий CPU

**PromQL** (пример для node_exporter):

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Правило:

- **Condition:** значение **above** порога, например **85** (проценты).
- **For:** **5m** — не алертить на секундный спайк.
- **Labels:** `severity=warning`, `team=platform`.

---

## Практика: сервис «лежит» (up)

```promql
up{job="payments-api"} == 0
```

- **For:** **1–2m** (учёт перезапуска пода).
- **Severity:** **critical**, отдельный contact point (PagerDuty/on-call).

---

## Практика: высокая латентность (p95)

```promql
histogram_quantile(
  0.95,
  sum by (le, job) (rate(http_request_duration_seconds_bucket{job="api"}[5m]))
) > 0.5
```

Порог **0.5** с — под ваш SLO; **For** **5–10m** при нестабильном трафике.

---

## Multi-condition (несколько условий)

В unified alerting часто делают **несколько запросов** (Query A, B, C) и условие:

- **И** (например: **ошибки высоки** **и** **трафик не нулевой**), чтобы не будить ночью из-за «деления на почти ноль».

Пример смысловой связки:

- **A:** `sum(rate(http_requests_total{job="api"}[5m])) > 10` — есть нагрузка.
- **B:** доля 5xx: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05`.

Оба должны выполняться за **один и тот же** интервал оценки.

---

## Recovery (автоматическое «закрытие»)

Grafana переводит алерт из **Firing** в **Normal**, когда условие перестаёт выполняться; в contact point можно настроить **resolve notification** (версия и чекбоксы зависят от релиза).

**Best practice:**

- явно проверить, что **on-call** получает **recovery** там, где это помогает (иногда шумит — тогда только firing);
- **`For`** и **pending** уменьшают **flapping**.

---

## Антипаттерны

| Плохо | Лучше |
|-------|--------|
| Алерт без **For** на метрике с шумом | Задержка 2–5–10 минут по типу сигнала |
| Один канал на всё | **Policies** по `severity` и команде |
| Десятки алертов «на всякий случай» | **SLO-ориентированный** минимальный набор + runbook |
| Дубли Grafana + Alertmanager без договорённости | Один «владелец» критичного пути |

---

## Чеклист production

| # | Практика |
|---|----------|
| 1 | **Runbook** в описании правила или ссылка в шаблоне уведомления |
| 2 | Секреты contact points вне Git |
| 3 | Тест правила: **Preview** / временный **Test contact point** |
| 4 | Именование: `Team_Service_Symptom` в **`alertname` / кастомных лейблах** |
| 5 | Периодический аудит: что реально срабатывало за квартал |

---

## Дополнительные материалы

- [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/)
- [Notification policies](https://grafana.com/docs/grafana/latest/alerting/fundamentals/notifications/)
- [Silences](https://grafana.com/docs/grafana/latest/alerting/configure-notifications/create-silence/)
