# Prometheus
![Grafana[200]](./images/welcome.avif)

---
**Prometheus** — система мониторинга метрик во **временных рядах**: модель **pull**, язык запросов **PromQL**, встроенный **TSDB**, оповещения через **Alertmanager** и богатая экосистема **exporters**.

---

## Материалы

- [**1. База**](topic-1-basis.md) — что такое Prometheus и зачем он нужен, pull vs push и Pushgateway, TSDB, архитектура (Server, exporters, Alertmanager), метрики и лейблы; Docker Compose с `node_exporter` и cAdvisor, production-практики.
- [**2. PromQL**](topic-2-promql.md) — instant/range векторы, `rate`/`irate`/`increase`, агрегация `by`/`without`, селекторы; error rate, saturation, histogram_quantile; примеры CPU/memory/RPS/5xx для Kubernetes и HTTP.
- [**3. Экспортеры и кастомные метрики**](topic-3-exporters-and-custom-metrics.md) — node/blackbox/postgres/nginx exporters; пример blackbox `relabel_configs`; кастомные counter/gauge/histogram на Go и Python; production-практики.
- [**4. Алертинг (Alertmanager)**](topic-4-alerting-and-alertmanager.md) — alert/recording rules, жизненный цикл, routing, grouping, inhibition, silencing; примеры High CPU, рестарт Pod, доля 5xx; Slack, email, Telegram; `promtool`/`amtool`.
- [**5. Работа с Kubernetes**](topic-5-kubernetes.md) — kube-prometheus-stack, ServiceMonitor/PodMonitor, kube-state-metrics и cAdvisor, кардинальность лейблов; Helm и пример манифестов.
- [**6. Grafana (визуализация)**](topic-6-grafana-visualization.md) — datasource, дашборды, переменные, Grafana vs Prometheus alerting; панели для ноды, приложения и Kubernetes.
- [**7. Производительность и масштабирование**](topic-7-performance-and-scaling.md) — cardinality, retention и TSDB, remote write, federation, шардирование, Thanos/Mimir/VictoriaMetrics; примеры конфигов и практика.
- [**8. Лучшие практики**](topic-8-best-practices.md) — именование (`*_total`, единицы), counter/gauge/histogram, гигиена лейблов, recording rules; примеры и `promtool`.
- [**9. Устранение неполадок**](topic-9-troubleshooting.md) — target DOWN, «дыры» в данных, медленный PromQL, память и кардинальность; `kubectl`/`curl`, метрики, лабораторные поломки.
- [**10. Production-кейсы**](topic-10-production-cases.md) — микросервисы (RED), SLI/SLO и error budget, Golden Signals; примеры PromQL и recording rules.

