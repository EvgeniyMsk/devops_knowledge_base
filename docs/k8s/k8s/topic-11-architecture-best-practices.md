# 11. Архитектура и Best Practices (уровень Middle+ / Senior)

---
Стратегия namespace’ов, мультикластер, Blue/Green и Canary, оптимизация затрат, обновление кластера, Disaster Recovery. Примеры манифестов с комментариями и best practices для production.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Namespace (стратегия)** | Виртуальное разделение ресурсов в кластере; выбор стратегии (по окружению, по команде, гибрид) влияет на изоляцию, RBAC, квоты и NetworkPolicy. |
| **ResourceQuota** | Ограничение суммарных ресурсов (CPU, память, число объектов) в namespace; не даёт одному namespace исчерпать кластер. |
| **LimitRange** | Ограничение и значения по умолчанию для requests/limits контейнеров в namespace; задаёт min/max и default. |
| **Multi-cluster** | Несколько независимых кластеров; цели: изоляция prod, геораспределение, соответствие требованиям; управление через GitOps/CI. |
| **Blue/Green** | Два идентичных окружения (Blue и Green); переключение трафика с одного на другое одномоментно; откат — переключить обратно. |
| **Canary** | Постепенный выкат: небольшая доля трафика на новую версию, затем увеличение при успешных метриках; снижает риски релиза. |
| **Disaster Recovery (DR)** | Восстановление после сбоя: бэкапы (etcd, данные приложений), план восстановления, RTO/RPO; тестирование восстановления. |
| **RTO (Recovery Time Objective)** | Целевое время восстановления сервиса после сбоя. |
| **RPO (Recovery Point Objective)** | Целевая точка восстановления по данным (допустимая потеря данных по времени). |

---

## Multi-namespace стратегия

**Зачем несколько namespace’ов:** изоляция по командам/приложениям/окружениям, ограничение ресурсов (ResourceQuota), RBAC по namespace, сетевая изоляция (NetworkPolicy), упрощение навигации и прав.

**Типичные подходы:**

| Подход | Описание | Когда использовать |
|--------|----------|---------------------|
| **Namespace по окружению** | `dev`, `staging`, `production` | Разделение dev/prod; разные квоты и политики. |
| **Namespace по приложению/команде** | `team-a`, `app-frontend`, `app-backend` | Изоляция команд, RBAC и квоты по приложению. |
| **Гибрид** | `production-app1`, `production-app2`, `staging` | Крупные команды, несколько приложений в production. |

### Пример: ResourceQuota и LimitRange на namespace

```yaml
# Ограничение потребления в namespace (например, для staging — меньше ресурсов, чем в production).
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    count/deployments.apps: "20"
---
# LimitRange задаёт дефолтные и максимальные requests/limits в namespace.
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-alpha
spec:
  limits:
    - default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
      type: Container
```

RBAC привязывать к namespace через Role + RoleBinding; для кросс-namespace доступа — ClusterRole + RoleBinding в каждом namespace. См. [5. Безопасность](topic-5-security.md).

---

## Multi-cluster

**Зачем несколько кластеров:** изоляция окружений (prod в отдельном кластере), геораспределение, соблюдение требований (данные не выходят из региона), отказоустойчивость. Альтернатива — федерация (KubeFed и др.), но чаще используют **несколько независимых кластеров** и единую точку управления (GitOps, CI).

### GitOps: один репозиторий, несколько кластеров

Манифесты или Helm-чарты хранятся в Git; в каждом кластере свой набор Argo CD Application’ов (или один Argo CD с разными destination cluster/namespace). Кластер задаётся в `destination.server` (URL API) или по имени в Argo CD.

```yaml
# Application для кластера production (cluster URL задаётся в Argo CD clusters).
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production-cluster-a
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://git.example.com/platform/apps.git
    path: apps/myapp
    helm:
      valueFiles:
        - values-production.yaml
  destination:
    server: https://cluster-a.example.com
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Разделение по кластерам: отдельная директория или ветка в Git (например, `clusters/prod-eu/`, `clusters/prod-us/`) и Application per cluster. Best practice: один источник истины в Git, минимум ручных отличий между кластерами (конфиг через values, не копипаста манифестов).

---

## Blue/Green и Canary

### Blue/Green

Два идентичных окружения (Blue и Green); трафик переключается с одного на другое одномоментно (переключение Service selector или Ingress backend). Откат — переключить обратно.

**Реализация в Kubernetes:** два Deployment с разными label’ами (например, `version: blue` и `version: green`). Service указывает на активную версию; переключение — смена `selector` в Service или использование двух Service и переключение на уровне Ingress.

```yaml
# Blue и Green — два Deployment с разными версиями образа и метками.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: app
          image: myapp:v1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: app
          image: myapp:v2
---
# Service переключается между blue и green сменой selector (version: blue или version: green).
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
    version: blue
  ports:
    - port: 80
      targetPort: 8080
```

Переключение на green: изменить `spec.selector.version` на `green` и применить манифест (или через Helm/Argo CD). Откат — вернуть `version: blue`.

### Canary

Часть трафика идёт на новую версию, остальная — на старую; при отсутствии ошибок долю новой версии увеличивают. Реализация: **Ingress с весами** (NGINX Ingress canary annotations) или **Argo Rollouts / Flagger** (CRD для canary/blue-green).

**Пример: Canary через аннотации NGINX Ingress**

```yaml
# Основной Ingress — стабильная версия.
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-main
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-stable
                port:
                  number: 80
---
# Canary Ingress — часть трафика по заголовку или по весу (canary-weight).
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-canary
                port:
                  number: 80
```

Увеличение канареечного трафика: изменить `canary-weight` на 50, затем 100; при проблемах — вернуть 0 или удалить canary Ingress. Для автоматизации по метрикам (ошибки, latency) используют Argo Rollouts или Flagger.

---

## Cost optimization

- **Requests и limits:** задавать адекватные values; завышенные limits раздувают планирование и не дают кластеру плотно упаковать поды. Использовать VPA для рекомендаций.
- **HPA и Cluster Autoscaler:** уменьшать неиспользуемые реплики и ноды; задавать разумные min/max.
- **Размер нод:** правый размер инстансов под нагрузку (не держать большие ноды ради одного маленького пода); при необходимости — несколько пулов нод (разные размеры).
- **Spot / Preemptible ноды:** для не критичных к выселению workload’ов снижают стоимость; использовать PDB и несколько реплик.
- **Автоотключение dev/staging:** выключать или масштабировать до нуля неиспользуемые окружения по расписанию (KEDA CronScaler, скрипты).

### Пример: Namespace с лимитами для контроля затрат

```yaml
# Квота ограничивает суммарное потребление в namespace; scheduler не запланирует поды сверх квоты.
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cost-cap
  namespace: staging
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

Мониторинг затрат: метрики по namespace (requests/usage), экспорт в биллинг провайдера; алерты при превышении бюджета.

---

## Cluster upgrades

- **Версии Kubernetes:** следовать совместимости (skew) между версиями kubelet, control plane и т.д.; обновлять по одной минорной версии (например, 1.27 → 1.28).
- **Порядок в self-managed:** сначала обновить control plane (по одной ноде при нескольких репликах), затем kubelet на worker-нодах. Drain нод перед обновлением; учитывать PDB.
- **Managed-кластеры (EKS, GKE, AKS):** провайдер обновляет control plane; worker-ноды обновляются через замену node pool или in-place upgrade. Настроить окно обновлений и тестировать в staging.
- **Проверка перед апгрейдом:** deprecated API (например, removal в 1.28+); тесты в staging на той же версии; бэкапы etcd и критичных данных.

```bash
# Проверка deprecated API (kubectl 1.27+)
kubectl get --raw /openapi/v3 | jq '.paths | keys'  # или использовать kubectl-convert
# Drain ноды перед обновлением
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

---

## Disaster Recovery

**Цели:** RTO (время восстановления) и RPO (потеря данных). План DR включает бэкапы, процедуры восстановления и регулярные проверки.

### etcd

- **Регулярные снапшоты etcd** (см. [2. Архитектура](topic-2-architecture.md)); хранить вне кластера с retention. Восстановление control plane из снапшота при потере кворума.
- **Managed-кластеры:** бэкапы control plane обычно обеспечивает провайдер; уточнить RPO/RTO.

### Приложения и данные

- **StatefulSet и тома:** снапшоты томов (VolumeSnapshot), бэкапы БД (дампы в объектное хранилище). Восстановление: новый PVC из снапшота или восстановление дампа в новый под.
- **Манифесты и конфигурация:** всё в Git (GitOps); восстановление кластера — установить Argo CD/Flux и указать на репозиторий; приложения развернутся из Git.
- **Runbook:** документированные шаги восстановления кластера с нуля, восстановления etcd, восстановления приложений из бэкапов; регулярные учения (DR drill).

### Чеклист DR

- [ ] Снапшоты etcd (или контроль plane у провайдера) с заданным RPO.
- [ ] Бэкапы томов и/или дампы БД; хранение вне кластера.
- [ ] Манифесты и конфигурация в Git; возможность развернуть кластер из Git.
- [ ] Документированный runbook восстановления; назначенные ответственные.
- [ ] Периодическая проверка восстановления (restore test).

---

## Best practices (сводка)

| Область | Рекомендация |
|--------|--------------|
| **Namespace** | Разделение по окружению/команде; ResourceQuota и LimitRange; RBAC по namespace. |
| **Multi-cluster** | Один источник истины в Git; минимум расхождений между кластерами; GitOps per cluster. |
| **Деплой** | Blue/Green или Canary для снижения риска; откат за счёт переключения трафика. |
| **Затраты** | Адекватные requests/limits, HPA, CA, правый размер нод, spot для некритичных workload’ов. |
| **Обновления** | По одной минорной версии; drain и PDB; тесты в staging; проверка deprecated API. |
| **DR** | Бэкапы etcd и данных; конфигурация в Git; runbook и учения. |

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Namespace с квотами** | Предсказуемые лимиты и изоляция по команде/окружению. |
| **Git как источник истины для всех кластеров** | Воспроизводимость и быстрый restore. |
| **Canary/Blue-Green** | Снижение риска при релизах. |
| **Регулярные DR-учения** | Подтверждение, что восстановление реально работает. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Всё в default namespace** | Нет изоляции, сложно управлять правами и квотами. | Выделять namespace по окружению/приложению. |
| **Ручные отличия между кластерами** | Дрейф конфигурации, сложный rollback. | Параметризация через values, один репо. |
| **Big-bang деплой без отката** | Высокий риск при баге в новой версии. | Blue/Green или Canary с быстрым откатом. |
| **Бэкапы без проверки восстановления** | В момент инцидента restore может не сработать. | Периодически восстанавливать из бэкапа в тестовый кластер. |

---

## Дополнительные материалы

- [Kubernetes — Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Argo CD — Multiple Clusters](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple-clusters/)
- [Argo Rollouts](https://argoproj.github.io/rollouts/)
- [NGINX Ingress — Canary](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary)
- [2. Архитектура](topic-2-architecture.md) — control plane, etcd backup
- [8. Автомасштабирование и отказоустойчивость](topic-8-autoscaling-resilience.md) — HPA, PDB
- [9. CI/CD и GitOps](topic-9-cicd-gitops.md) — GitOps и Argo CD
- [10. Отладка и эксплуатация](topic-10-troubleshooting-production.md) — сценарии и runbook
