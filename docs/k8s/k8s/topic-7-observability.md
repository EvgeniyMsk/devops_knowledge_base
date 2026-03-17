# 7. Наблюдаемость в Kubernetes (Observability)

---
Мониторинг (Prometheus, kube-state-metrics, Node Exporter), алертинг (Alertmanager, SLO/SLA/SLI), логирование (EFK/ECK/Loki, best practices для stdout), трейсинг (OpenTelemetry, Jaeger/Tempo). Примеры манифестов с комментариями и best practices для production.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Observability (наблюдаемость)** | Возможность понимать состояние системы по внешним данным: метрики, логи, трейсы; в Kubernetes — состояние нод, подов, приложений. |
| **Метрики** | Числовые показатели во времени (CPU, память, RPS, latency); собираются по pull (Prometheus) или push; хранятся во временных рядах. |
| **Prometheus** | Система мониторинга: сбор метрик по HTTP (scrape), хранение во временных рядах, запросы PromQL, алертинг. |
| **kube-state-metrics** | Сервис, экспортирующий состояние объектов Kubernetes (поды, деплойменты, ноды и т.д.) в формате метрик Prometheus. |
| **Node Exporter** | Экспортер метрик ноды (CPU, память, диск, сеть) в формате Prometheus; обычно запускается как DaemonSet. |
| **ServiceMonitor** | CRD Prometheus Operator: указывает Prometheus, какой Service скрейпить и по какому пути/порту для сбора метрик. |
| **PrometheusRule** | CRD Prometheus Operator: группы правил алертинга и записи; при срабатывании условия генерируется алерт. |
| **Alertmanager** | Компонент Prometheus: получает алерты, группирует, дедуплицирует и отправляет в каналы (Slack, PagerDuty, email). |
| **SLI (Service Level Indicator)** | Метрика качества сервиса (доля успешных запросов, latency p99 и т.д.). |
| **SLO (Service Level Objective)** | Целевое значение SLI (например, 99.9% доступности); основа для алертов и error budget. |
| **SLA (Service Level Agreement)** | Договор с пользователями с последствиями при нарушении; алерты настраивают так, чтобы реагировать до нарушения SLO. |
| **Трейсинг (distributed tracing)** | Отслеживание пути запроса через несколько сервисов (spans, trace ID); инструменты: Jaeger, Tempo, OpenTelemetry. |
| **OpenTelemetry** | Стандарт и инструменты для сбора телеметрии (метрики, логи, трейсы) и экспорта в бэкенды. |

---

## Обзор: зачем observability в Kubernetes

Наблюдаемость складывается из **метрик** (что и как работает), **логов** (события и ошибки) и **трейсов** (путь запроса по сервисам). В кластере важно отслеживать состояние нод, подов, контроллеров и приложений, чтобы быстро обнаруживать сбои и деградации. Ниже — типовой стек и практические примеры.

---

## Мониторинг: Prometheus, kube-state-metrics, Node Exporter

**Prometheus** — сбор метрик по pull-модели (скрейпит HTTP-эндпоинты). В Kubernetes часто разворачивают **Prometheus Operator**: он управляет объектами `Prometheus`, `ServiceMonitor`, `PrometheusRule`. Метрики кластера дают **kube-state-metrics** (состояние объектов API: поды, деплойменты, ноды и т.д.) и **Node Exporter** (метрики ноды: CPU, память, диск, сеть). Prometheus собирает их по ServiceMonitor’ам и хранит во временных рядах; запросы — PromQL, визуализация — обычно Grafana.

### ServiceMonitor: сбор метрик с kube-state-metrics

```yaml
# ServiceMonitor указывает Prometheus, какой Service скрейпить и по какому пути.
# Prometheus Operator создаёт конфигурацию таргетов из ServiceMonitor'ов (selector по меткам).
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  namespaceSelector:
    matchNames:
      - monitoring
  endpoints:
    - port: http-metrics
      interval: 30s
      path: /metrics
```

### ServiceMonitor: Node Exporter (на каждой ноде)

```yaml
# Node Exporter обычно запускается как DaemonSet; Service агрегирует поды по нодам.
# Prometheus скрейпит каждый под (или один сервис с одной репликой на ноду — зависит от установки).
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: node-exporter
  namespaceSelector:
    any: true
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### Пример Pod для приложения с метриками (instrumentation)

```yaml
# Приложение отдаёт метрики в формате Prometheus на :8080/metrics.
# Аннотации не обязательны; используются некоторыми инструментами для автообнаружения.
apiVersion: v1
kind: Pod
metadata:
  name: app-with-metrics
  labels:
    app: myapp
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  containers:
    - name: app
      image: myapp:latest
      ports:
        - containerPort: 8080
          name: metrics
```

---

## Алертинг: Alertmanager и правила

**Alertmanager** получает алерты от Prometheus (по правилам в **PrometheusRule**), группирует, дедуплицирует и отправляет в каналы (Slack, PagerDuty, email и т.д.). **PrometheusRule** — CRD от Prometheus Operator: список групп с правилами; при срабатывании условие генерирует алерт.

### SLO / SLA / SLI (кратко)

- **SLI** (Service Level Indicator) — метрика доступности/качества (например, доля успешных запросов, latency p99).
- **SLO** (Service Level Objective) — целевое значение SLI (например, 99.9% доступности).
- **SLA** (Service Level Agreement) — договор с пользователями с последствиями при нарушении. Алерты часто настраивают так, чтобы срабатывать до нарушения SLO (burn rate, error budget).

---

## Примеры манифестов: алерты (PrometheusRule)

### Алерт: Pod в CrashLoopBackOff

```yaml
# При переходе пода в CrashLoopBackOff нужно реагировать быстро — приложение недоступно.
# kube_pod_container_status_waiting_reason — метрика от kube-state-metrics.
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-app-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: kubernetes.pods
      interval: 30s
      rules:
        - alert: PodCrashLooping
          expr: |
            kube_pod_container_status_waiting_reason{
              reason="CrashLoopBackOff"
            } == 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} in CrashLoopBackOff"
            description: "Container {{ $labels.container }} has been in CrashLoopBackOff for more than 1 minute."
```

### Алерт: нода не Ready

```yaml
# Нода в состоянии NotReady — поды на ней могут не планироваться или быть недоступны.
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-nodes-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: kubernetes.nodes
      interval: 30s
      rules:
        - alert: NodeNotReady
          expr: |
            kube_node_status_condition{
              condition="Ready",
              status="true"
            } == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} is NotReady"
            description: "Node has been NotReady for more than 2 minutes. Check kubelet and node resources."
```

### Дополнительные полезные алерты (фрагмент)

```yaml
# Примеры: высокое использование памяти на ноде, поды не готовы.
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-extra-alerts
  namespace: monitoring
spec:
  groups:
    - name: kubernetes.resources
      rules:
        - alert: NodeMemoryHighUsage
          expr: |
            (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage on node {{ $labels.instance }}"
        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_spec_replicas != kube_deployment_status_replicas_available
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has replica mismatch"
```

---

## Alertmanager: базовая конфигурация

```yaml
# ConfigMap с конфигом Alertmanager: маршруты и приёмники (receiver).
# В production подставляют реальный Slack webhook / PagerDuty key и т.д.
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'default'
      routes:
        - match:
            severity: critical
          receiver: critical
          continue: true
    receivers:
      - name: default
        # slack_configs / email / pagerduty — по необходимости
      - name: critical
        # Отправка в канал для срочных алертов
```

---

## Логирование: EFK / ECK / Loki и stdout

**EFK** (Elasticsearch, Fluentd, Kibana) и **ECK** (Elastic Cloud on Kubernetes — оператор для Elasticsearch/Kibana в K8s) — классический стек: сбор логов из подов/нод, индексация в Elasticsearch, поиск в Kibana. **Loki** — хранилище логов, оптимизированное под метки и интеграцию с Grafana; часто используется вместе с Prometheus.

**stdout best practices для контейнеров:**

- Писать логи в **stdout/stderr**, не в файлы внутри контейнера — так их собирает kubelet и любой лог-коллектор (Fluent Bit, Fluentd, DaemonSet).
- Использовать **структурированные логи** (JSON) — проще парсить и фильтровать.
- Не логировать чувствительные данные (пароли, токены) и минимизировать объём в production (уровень info/warn в норме, debug — по флагу или в dev).

---

## Трейсинг: OpenTelemetry, Jaeger, Tempo

**OpenTelemetry** — единый стандарт для метрик, логов и трейсов: SDK в приложении, экспорт в коллектор (OTLP). **Jaeger** — классический бэкенд для трейсов (сбор, хранение, UI). **Tempo** — бэкенд трейсов из экосистемы Grafana, хорошо интегрируется с Loki и Prometheus. В production приложение инструментируют (автоинструментация или ручные спаны), экспортируют трейсы в коллектор; коллектор отдаёт их в Jaeger/Tempo.

Конкретные манифесты зависят от выбранного стека (Helm-чарты Jaeger, OpenTelemetry Operator и т.д.); базовый сценарий — Deployment коллектора и приём OTLP на порту.

---

## Best practices и примеры из production

### Мониторинг

- Включить **kube-state-metrics** и **Node Exporter**; собирать их через ServiceMonitor. Без них видимость состояния деплойментов, подов и нод ограничена.
- Для приложений — **метрики в формате Prometheus** на отдельном порту или пути; не смешивать с основным API. Задавать **requests/limits** для подов мониторинга, чтобы они не конкурировали с workload’ами.
- **Хранить метрики с разумным retention** и при необходимости использовать remote write (например, в долгоживущее хранилище) для долгосрочного анализа.

### Алертинг

- **Алерты должны быть действиями:** каждый алерт — кому и что делать (runbook, ссылка на playbook). Избегать «шумных» правил: настроить `for`, группировку и дедупликацию в Alertmanager.
- **Критичность:** critical — реагировать сразу (PodCrashLooping, NodeNotReady); warning — в рабочее время или следующий он-колл. Разделять маршруты в Alertmanager по severity.
- **SLO-based alerting:** где есть SLO (доступность, latency), настраивать алерты по error budget / burn rate, а не только по «ошибки > 0».

### Логирование

- **Только stdout:** не писать логи в файлы в контейнере; иначе при падении пода логи теряются и сбор усложняется.
- **Структура (JSON)** и единый формат (поля level, message, trace_id при наличии) упрощают поиск и связку с трейсами.
- Ограничивать объём и не логировать персональные/секретные данные; при необходимости маскировать на стороне коллектора.

### Трейсинг

- В распределённых сервисах **trace_id** в логах и метриках позволяет связать запрос с трейсом в Jaeger/Tempo. Единый формат (W3C Trace Context, OpenTelemetry) упрощает передачу между сервисами.

---

## Практика: написать алерты PodCrashLooping и NodeNotReady

1. Убедиться, что в кластере установлены Prometheus (или Prometheus Operator), kube-state-metrics и при необходимости Node Exporter.
2. Создать **PrometheusRule** с двумя группами правил:
   - **PodCrashLooping:** выражение по `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1`, `for: 1m`, severity critical.
   - **NodeNotReady:** выражение по `kube_node_status_condition{condition="Ready",status="true"} == 0`, `for: 2m`, severity critical.
3. Проверить, что Prometheus подхватил правило (`prometheus_rules` в Prometheus UI или статус PrometheusRule в кластере).
4. Имитировать условие (например, перевести под в CrashLoopBackOff или отключить kubelet на тестовой ноде) и убедиться, что алерт появляется в Prometheus и доходит до Alertmanager.
5. Настроить receiver в Alertmanager (Slack/email) для severity critical и проверить доставку.

Примеры полных PrometheusRule приведены выше в разделе «Примеры манифестов: алерты».

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Метрики приложения в Prometheus-формате** | Единый стек, дашборды и алерты в одном месте. |
| **ServiceMonitor для всех целевых сервисов** | Автоматический сбор без ручного прописывания таргетов. |
| **Алерты с for и понятными аннотациями** | Меньше ложных срабатываний, быстрая реакция по summary/description. |
| **Логи только в stdout, JSON** | Упрощение сбора и корреляции с трейсами. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Логи в файлы внутри контейнера** | Потеря при падении пода, сложный сбор. | stdout/stderr, лог-коллектор на ноде. |
| **Слишком много алертов без группировки** | Усталость от уведомлений, игнор. | Группировка, дедупликация, runbook, приоритеты. |
| **Мониторинг без алертов на состояние кластера** | Сбои обнаруживаются пользователями. | Как минимум алерты на CrashLoopBackOff, NodeNotReady, replica mismatch. |

---

## Дополнительные материалы

- [Prometheus](https://prometheus.io/docs/)
- [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [OpenTelemetry](https://opentelemetry.io/)
- [Grafana Loki](https://grafana.com/oss/loki/)
- [Kubernetes — Observability](https://kubernetes.io/docs/concepts/cluster-administration/monitoring/)
