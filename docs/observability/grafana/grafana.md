# Grafana

![Grafana[200]](./images/welcome.jpeg)


---
**Grafana** — визуализация **метрик**, **логов** и **трейсов**, дашборды для операторов и разработчиков, алерты и единый вход к **Prometheus**, **Loki**, **Elasticsearch**, **InfluxDB** и десяткам других источников. В материалах ниже — от основ до production-паттернов.

---

## Материалы

- [**1. База**](topic-1-basis.md) — роль Grafana в Observability, metrics vs logs vs traces, datasource’ы, Docker с Prometheus и node_exporter, первый дашборд (CPU, память), провижининг.
- [**2. Дашборды и визуализация**](topic-2-dashboards.md) — панели Time series / Stat / Table, переменные и зависимости, transformations, drill-down; Node Exporter и Kubernetes-обзор с фильтром namespace/pod.
- [**3. Prometheus и PromQL в Grafana**](topic-3-prometheus-promql.md) — `rate`/`irate`, `sum by`/`avg by`, `histogram_quantile`, кардинальность, recording rules; RPS, latency p95/p99, error rate, дашборд SLI/SLO.
- [**4. Алертинг**](topic-4-alerting.md) — unified alerting, Grafana vs Prometheus/Alertmanager, contact points, silence/mute; CPU, down, latency, multi-condition, recovery.
- [**5. Grafana и Kubernetes**](topic-5-kubernetes.md) — Helm (`kube-prometheus-stack`, чарт Grafana), провижининг datasource/дашбордов, sidecar ConfigMap, RBAC и multi-tenancy; дашборды кластера и приложения.
- [**6. Логи и трассы**](topic-6-logs-traces.md) — Loki, Tempo, связка логи/метрики/трассы; LogQL, провижининг, data links и exemplars; drill-down.
- [**7. Провижининг и GitOps**](topic-7-provisioning-gitops.md) — YAML datasources/dashboards, HTTP API, Terraform; дашборды в Git, CI/CD (GitHub Actions / GitLab CI).
- [**8. Безопасность и доступы**](topic-8-security-access.md) — пользователи, Teams, роли Viewer/Editor/Admin; OAuth GitHub/Google; права на папки; разделение Dev/Ops/Viewer.
- [**9. Производительность и практики**](topic-9-performance-practices.md) — оптимизация запросов, кардинальность, дизайн дашбордов; снижение нагрузки на Prometheus, recording rules.
- [**10. Production-кейсы**](topic-10-production-scenarios.md) — деградация сервиса, spike latency, алерты без шума, RCA, SLO-дашборд для бизнеса.
