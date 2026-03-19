# 5. Работа с Kubernetes

---
Для **Middle+ DevOps** стек почти всегда включает **Prometheus Operator** и чарт **`kube-prometheus-stack`**: из коробки получаются **Prometheus**, **Alertmanager**, **node_exporter**, **kube-state-metrics**, интеграция с **cAdvisor**/kubelet и CRD **`ServiceMonitor` / `PodMonitor`**. Ниже — как это связано, влияние **лейблов** на **кардинальность** и практика развёртывания через **Helm**.

---

## kube-prometheus-stack (Helm)

Чарт **`kube-prometheus-stack`** объединяет:

- **Prometheus Operator** (управление `Prometheus`, `ServiceMonitor`, правилами и т.д.);
- **Prometheus** + **Alertmanager** (+ часто **Grafana**);
- **node_exporter** (DaemonSet);
- **kube-state-metrics** (состояние объектов API);
- настройки для сбора метрик **kubelet/cAdvisor** (контейнеры).

Best practices:

- отдельный **`Namespace`** (`monitoring`) и **RBAC** по принципу минимума;
- **storageClass** и размер томов под TSDB заранее;
- **Ingress** или **Gateway** к Grafana/UIs только с SSO/TLS;
- `values.yaml` под ваш кластер: ретенция, ресурсы, **allowLists** для scrape.

### Пример установки (ориентир)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values-prod.yaml
```

Комментарий: имя релиза (`kube-prom`) часто используют в **`release`-лейбле** для селекторов `ServiceMonitor` — см. ваш `values` (`prometheus.prometheusSpec.serviceMonitorSelector`).

---

## ServiceMonitor и PodMonitor

Оператор смотрит на CR и добавляет **цели** в конфигурацию Prometheus.

| Ресурс | Когда использовать |
|--------|-------------------|
| **ServiceMonitor** | Приложение уже **обслуживается Service** с портом метрик (стандартный путь) |
| **PodMonitor** | Нужен scrape **Pod** напрямую (headless, sidecar, без стабильного Service) |

### ServiceMonitor: пример для приложения

Предположим: `Service` с лейблом `app: payments-api` и именем порта **`metrics`**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-api
  labels:
    app: payments-api
spec:
  selector:
    app: payments-api
  ports:
    - name: metrics
      port: 8080
      targetPort: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payments-api
  namespace: production
  labels:
    release: kube-prom
spec:
  selector:
    matchLabels:
      app: payments-api
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: metrics
      interval: 30s
      scrapeTimeout: 10s
      path: /metrics
```

**`release: kube-prom`** — типичный лейбл, который ожидает селектор Prometheus из чарта (значение должно **совпасть** с вашим релизом; проверьте `kubectl get prometheus -n monitoring -o yaml` → `serviceMonitorSelector`).

Best practice: узкий **`namespaceSelector`**; не ставить `any: true` в production без контроля.

### PodMonitor (фрагмент)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myjob-workers
  labels:
    release: kube-prom
spec:
  selector:
    matchLabels:
      app: worker
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

---

## Метрики Kubernetes: kube-state-metrics и cAdvisor

| Источник | Что даёт |
|----------|-----------|
| **kube-state-metrics** | Снимок **желаемого/фактического** состояния: Deployment, Pod, Node, PVC… (`kube_pod_*`, `kube_deployment_*`…) |
| **cAdvisor / kubelet** | **Ресурсы контейнеров**: CPU, память, сеть, число рестартов (`container_*`) |

Комментарий: имена и набор лейблов зависят от версии **Kubernetes** и **cadvisor**; всегда сверяйте `metric_relabel_configs` в живом Prometheus.

---

## Лейблы в K8s и кардинальность

Каждый уникальный набор лейблов метрики — отдельный **time series**. В Kubernetes естественно высокая **частота смены** имён Pod (`ReplicaSet` hash), **UID**, IP.

| Риск | Практика |
|------|----------|
| Взрыв серий из **`pod`**, **`container_id`** | В запросах и алертах группировать **`deployment`**, **`namespace`**; в редких случаях — `relabel`/`metric_relabel_configs` для дропа лишнего |
| Пользовательские лейблы на Pod | Не класть в лейблы уникальные ID запросов |
| Множество пустых `Service` без селектора | Лишние цели scrape — чистить |

В **Helm values** у prometheus-operator часто настраивают **дроп** дорогих метрик или **sample limit** на job — используйте после анализа нагрузки на TSDB.

---

## Практика: развернуть Prometheus и подключить приложение

1. Установить **`kube-prometheus-stack`** в `monitoring` с вашим `values-prod.yaml`.
2. Убедиться, что **Prometheus** в статусе **Ready** и **targets** в UI показывают default jobs.
3. Добавить **порт метрик** в Pod/Deployment и **Service** с именем порта (`metrics`).
4. Создать **`ServiceMonitor`** с правильным **`release`** (или изменить селектор Prometheus под ваши лейблы).
5. В Prometheus UI → **Status → Targets** найти job с вашим сервисом в состоянии **UP**.

Проверка с CLI (если есть доступ к pod Prometheus):

```bash
kubectl -n monitoring port-forward svc/kube-prom-kube-prometheus-prometheus 9090:9090
# открыть http://localhost:9090/targets
```

Имена сервисов зависят от релиза — подставьте `kubectl get svc -n monitoring`.

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | Резервирование TSDB, алерты на **дис** и **WAL** |
| 2 | **NetworkPolicy**: кто может стучаться в `:9090` и в метрики приложений |
| 3 | Единая политика **`ServiceMonitor`** лейблов (`team`, `tier`) без лишней кардинальности |
| 4 | Версии чарта закрепить в Git (без «latest» в prod) |
| 5 | Отдельный **Grafana** datasource и backup дашбордов |

---

## Дополнительные материалы

- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Operator design](https://prometheus-operator.dev/docs/getting-started/introduction/)
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)

