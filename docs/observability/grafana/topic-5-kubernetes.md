# 5. Grafana и Kubernetes

---
В **production** Grafana почти всегда живёт в **кластере**: **Helm**, **провижининг** datasource’ов и дашбордов, **RBAC** Kubernetes и при необходимости **мультитенантность** внутри самой Grafana. Ниже — типовые паттерны, **`kube-prometheus-stack`** и отдельный чарт **`grafana`**, примеры **`values`** и идеи дашбордов **кластера** и **приложения**.

База по стеку мониторинга в K8s — в материале [Prometheus: работа с Kubernetes](../prometheus/topic-5-kubernetes.md).

---

## Два распространённых пути

| Подход | Когда удобен |
|--------|----------------|
| **`kube-prometheus-stack`** | Нужны **Prometheus Operator**, Alertmanager, экспортеры и **Grafana** «из одного релиза» |
| Чарт **`grafana`** (+ свой Prometheus) | Уже есть Prometheus вне чарта; нужна только **визуализация** или кастомный lifecycle Grafana |

**Best practice:** не плодить **две** Grafana на один кластер без осознанного разделения (например **platform** vs **tenant**).

---

## Helm: `kube-prometheus-stack` и Grafana

Релиз часто поднимают так (имена и namespace — ваши):

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values-monitoring.yaml
```

Фрагмент **`values-monitoring.yaml`** — Grafana и секрет пароля:

```yaml
grafana:
  enabled: true
  # Пароль админа из существующего Secret (рекомендуется вместо plain text)
  admin:
    existingSecret: grafana-admin-credentials
    userKey: admin-user
    passwordKey: admin-password

  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - grafana.example.com
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.example.com

  # Sidecar: подхватывать ConfigMap с дашбордами по лейблу
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      folder: /tmp/dashboards
    datasources:
      enabled: true
      label: grafana_datasource
```

**Production:** **Ingress** только с **TLS** и **SSO** (OAuth/OIDC) где возможно; пароль **admin** — ротация и сложность по политике.

---

## Провижининг: datasources и dashboards

### Datasource через ConfigMap (sidecar `grafana_datasource`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-ds
  namespace: monitoring
  labels:
    grafana_datasource: "1"
data:
  prometheus.yaml: |
    apiVersion: 1
    deleteDatasources:
      - name: Prometheus
        orgId: 1
    datasources:
      - name: Prometheus
        uid: prom-k8s
        type: prometheus
        access: proxy
        url: http://kube-prom-kube-prometheus-prometheus.monitoring.svc:9090
        isDefault: true
        jsonData:
          timeInterval: 30s
        editable: false
```

Имя сервиса Prometheus зависит от **имени Helm-релиза** (`kube-prom` в примере) — проверьте: `kubectl get svc -n monitoring`.

### Dashboards через ConfigMap (лейбл `grafana_dashboard`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-team-overview
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  team-overview.json: |
    { "uid": "team-overview", "title": "...", "panels": [] }
```

**Best practice:** большие JSON храните в **Git** и собирайте в образ/ConfigMap в CI; вручную править бой через UI без обратного экспорта — быстро приводит к **drift**.

---

## RBAC в Kubernetes и доступ к Grafana

| Уровень | Практика |
|---------|----------|
| **Pod ServiceAccount** | Минимальные права; Grafana редко нужен доступ к **Secret** всего кластера — избегайте «cluster-admin» для пода |
| **Ingress / Gateway** | Аутентификация на входе (OAuth2-proxy, корпоративный IdP) |
| **NetworkPolicy** | Ограничить откуда можно достучаться до `svc/grafana` |

---

## Multi-tenancy (организации и папки)

- **Organizations** в Grafana — изоляция дашбордов и datasource’ов между бизнес-юнитами (дороже в администрировании).
- Чаще на практике: **одна** организация, **папки** по командам + **RBAC Grafana** (Viewer/Editor), **разные** `namespace` в K8s для приложений.

**Production:** не смешивать **prod** и **non-prod** данные в одном datasource без фильтров и **переменных** по `cluster`/`environment`.

---

## Практика: дашборд кластера

1. После установки `kube-prometheus-stack` откройте встроенные дашборды (**Nodes**, **Kubernetes / Compute Resources / Cluster**).
2. Добавьте переменные **`cluster`** (если несколько кластеров в одной Grafana) или **`namespace`**.

Импорт по ID: **15757** (варианты меняются — ищите «Kubernetes cluster» в grafana.com/dashboards под ваш стек).

---

## Практика: дашборд приложения

1. Экспортируйте метрики приложения через **`ServiceMonitor`** (см. трек Prometheus) или Pod с аннотациями scrape.
2. Создайте дашборд с переменными **`namespace`**, **`pod`**, **`deployment`**.
3. Панели: **RPS**, **latency p95**, **ошибки** (как в [PromQL в Grafana](topic-3-prometheus-promql.md)).

---

## Отдельный чарт Grafana (кратко)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  -f values-grafana-standalone.yaml
```

Используют, когда Prometheus уже развёрнут (managed или другой релиз), а Grafana нужно обновлять/масштабировать независимо.

---

## Чеклист production

| # | Практика |
|---|----------|
| 1 | Пароли и API-ключи — **Secret**; **не** в plain `values` в Git |
| 2 | Провижининг datasource + дашборды = **GitOps** |
| 3 | Одна «истинная» Grafana на кластер или явное разделение ролей |
| 4 | Бэкап **БД Grafana** (SQLite/Postgres) при stateful сценарии |
| 5 | Версия чарта **закреплена** (не «latest» без теста) |

---

## Дополнительные материалы

- [Grafana Helm chart](https://github.com/grafana/helm-charts/tree/main/charts/grafana)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Run Grafana in Kubernetes](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/)
