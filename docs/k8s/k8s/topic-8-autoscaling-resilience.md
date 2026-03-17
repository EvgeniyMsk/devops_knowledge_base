# 8. Автомасштабирование и отказоустойчивость

Базовые концепции Pod и probes описаны в [3. Workloads и API-объекты](topic-3-workloads-api.md); здесь фокус на масштабировании и управлении распределением нагрузки.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **HPA (Horizontal Pod Autoscaler)** | Контроллер: изменяет число реплик Deployment/StatefulSet по метрикам (CPU, память или custom); периодически сравнивает текущее значение с целевым. |
| **VPA (Vertical Pod Autoscaler)** | Рекомендует или автоматически изменяет requests/limits подов на основе фактического потребления; режимы: Off, Initial, Recreate, Auto. |
| **Cluster Autoscaler** | Внешний компонент: увеличивает/уменьшает число нод в облачном кластере при нехватке ресурсов (pending поды) или при недостаточной загрузке. |
| **KEDA (Kubernetes Event-driven Autoscaling)** | Оператор: масштабирует workload по внешним событиям (очереди, Cron, HTTP, Prometheus); позволяет масштабировать до 0 реплик. |
| **ScaledObject** | CRD KEDA: привязывает триггер (источник и порог) к Deployment/StatefulSet для автомасштабирования по событиям. |
| **PodDisruptionBudget (PDB)** | Ограничение числа добровольно вытесняемых подов: minAvailable или maxUnavailable; защищает от одновременного обнуления реплик при обновлении нод. |
| **Affinity (сродство)** | Правила планирования: предпочтение нод/под с определёнными метками (nodeAffinity, podAffinity); притягивание подов к нодам или друг к другу. |
| **Anti-affinity** | Правила планирования: избегать размещения подов на одних нодах или с определёнными подами; для распределения и отказоустойчивости. |
| **Taint (пятно)** | Метка ноды, «отталкивающая» поды; под без подходящего toleration не будет запланирован на эту ноду. |
| **Toleration (допуск)** | Разрешение пода быть запланированным на ноду с заданным taint; используется для выделенных нод и DaemonSet. |
| **Eviction (вытеснение)** | Принудительное завершение пода на ноде (нехватка ресурсов, drain); порядок определяется QoS и приоритетом. |

---

## Автомасштабирование

### HPA (Horizontal Pod Autoscaler)

**HPA** изменяет количество реплик Deployment/StatefulSet и других масштабируемых ресурсов по метрикам (CPU, memory или custom/external). Периодически контроллер сравнивает текущее значение с целевым и корректирует `spec.replicas`. Для кастомных метрик нужен metrics-server (для CPU/memory) или адаптер (Prometheus и т.д.).

### Пример манифеста: HPA по CPU и памяти

```yaml
# HPA масштабирует Deployment myapp от 2 до 10 реплик по средней загрузке CPU (70%) и памяти (80%).
# Требуется metrics-server в кластере (для метрик resource).
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

- **behavior** — сглаживание: scaleDown не агрессивный (окно 5 мин, не более 50% реплик в минуту), scaleUp быстрый (до 100% за 15 с). Снижает «дрожание» при скачках нагрузки.

### HPA по кастомной метрике (Prometheus)

```yaml
# Масштабирование по метрике из Prometheus (требуется prometheus-adapter или custom metrics API).
# Пример: целевое значение 1000 RPS на реплику (приложение экспортирует http_requests_per_second).
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa-requests
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

---

### VPA (Vertical Pod Autoscaler)

**VPA** рекомендует или автоматически изменяет **requests/limits** подов (CPU, memory) на основе фактического потребления. Режимы: Off, Initial, Recreate, Auto. В production часто используют в режиме **Recommend** (только рекомендации для ручной настройки) или **Recreate** (VPA пересоздаёт поды с новыми ресурсами — возможны кратковременные простои). Требует установки VPA components (recommender, updater, admission controller).

### Пример: VPA для Deployment

```yaml
# VPA будет рекомендовать/применять ресурсы для подов, попадающих под selector.
# updatePolicy: Off — только рекомендации; Recreate — автоматическое обновление (пересоздание подов).
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Recreate"
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
```

---

### Cluster Autoscaler

**Cluster Autoscaler** (внешний компонент, не манифест в кластере) увеличивает/уменьшает число нод в облачном кластере: добавляет ноды, когда поды не могут быть запланированы из‑за нехватки ресурсов (pending), и удаляет ноды, когда они становятся недостаточно загружены. Настраивается под провайдера (AWS, GCP, Azure и т.д.). Для корректной работы поды не должны «прилипать» к нодам без необходимости; DaemonSet’ы и поды с local storage учитываются при решении об удалении ноды.

---

### KEDA (Kubernetes Event-driven Autoscaling)

**KEDA** масштабирует workload’ы (вплоть до 0 реплик) по внешним событиям: очереди (RabbitMQ, Kafka), Cron, HTTP, метрики из Prometheus и др. Устанавливается оператор; создаётся объект **ScaledObject**, привязываемый к Deployment/StatefulSet и указывающий trigger (источник и порог). Подходит для очередей и событийно-ориентированных задач.

### Пример: KEDA ScaledObject для очереди

```yaml
# KEDA масштабирует consumer от 0 до 10 реплик по длине очереди RabbitMQ.
# Требуется установленный KEDA и секрет с подключением к RabbitMQ.
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-consumer-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rabbitmq-consumer
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: rabbitmq
      metadata:
        protocol: amqp
        host: amqp://guest:password@rabbitmq.default.svc.cluster.local:5672
        queueName: tasks
        queueLength: "5"
```

---

## Устойчивость: Probes, PDB, размещение

### Liveness / Readiness / Startup probes

- **Liveness** — жив ли контейнер; при неуспехе kubelet перезапускает контейнер.
- **Readiness** — готов ли принимать трафик; при неуспехе под исключается из Endpoints (Service не шлёт трафик).
- **Startup** — для медленного старта: пока startup не успешен, liveness/readiness не проверяются. Избегает перезапуска контейнера во время инициализации.

Подробнее и примеры — в [3. Workloads и API-объекты](topic-3-workloads-api.md). Ниже — production-ориентированный фрагмент с таймингами.

### Пример: все три probe с разумными таймингами

```yaml
# Медленно стартующее приложение: startup даёт до 60s на запуск, затем работают liveness/readiness.
# failureThreshold * periodSeconds = максимальное время до перезапуска/исключения из Service.
containers:
  - name: app
    image: myapp:latest
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 2
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2
```

---

### PodDisruptionBudget (PDB)

**PodDisruptionBudget** ограничивает, сколько подов данного приложения могут быть одновременно недоступны при *добровольных* сбоях (eviction при drain ноды, обновление узла). Задаётся через `minAvailable` (минимум доступных подов) или `maxUnavailable` (максимум недоступных). Контроллер eviction учитывает PDB: при drain он не выселит поды, если это нарушит PDB (пока не освободятся другие). Для отказоустойчивости при обновлениях нод PDB обязателен.

### Примеры манифестов: PDB

```yaml
# Минимум 2 пода из приложения app должны быть доступны при добровольном eviction.
# Подходит для Deployment с 3+ репликами.
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: default
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
---
# Альтернатива: не более 1 пода может быть недоступно одновременно.
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb-maxunavailable
  namespace: default
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

### PDB для StatefulSet (одна реплика БД)

```yaml
# Для single-replica StatefulSet: разрешаем 0 maxUnavailable — eviction не будет выселять единственный под.
# Обновление ноды тогда потребует ручного вмешательства или временного отключения PDB.
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: production
spec:
  maxUnavailable: 0
  selector:
    matchLabels:
      app: postgres
```

---

### Affinity и Anti-affinity

**Node affinity** — планирование подов на ноды с определёнными метками (например, тип инстанса, зона). **Pod affinity** — размещать под рядом с другими подами (например, в той же зоне). **Pod anti-affinity** — разносить поды по разным нодам/зонам для отказоустойчивости (preferred или required).

### Пример: разнести реплики по зонам (anti-affinity)

```yaml
# Предпочтительно размещать поды app в разных зонах (topologyKey: topology.kubernetes.io/zone).
# preferredDuringSchedulingIgnoredDuringExecution — мягкое предпочтение; при нехватке нод поды могут оказаться в одной зоне.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: myapp
                topologyKey: topology.kubernetes.io/zone
```

### Жёсткое anti-affinity: реплики на разных нодах

```yaml
# requiredDuringSchedulingIgnoredDuringExecution — под не будет запланирован, если не найдётся нода без другого пода с app=myapp.
# Гарантирует распределение по нодам (при достаточном числе нод).
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname
```

---

### Taints и Tolerations

**Taint** на ноде отталкивает поды, у которых нет соответствующего **toleration**. Используется для выделения нод под специальные workload’ы (например, только GPU, только системные поды) или для защиты нод от планирования обычных подов. **Toleration** в поде разрешает планирование на ноды с указанным taint (по key, value, effect).

### Пример: выделенные ноды для тяжёлых задач

```yaml
# Нода помечена taint (делается вручную или через настроенный node pool):
# kubectl taint nodes node-1 workload=heavy:NoSchedule
---
# Под с toleration планируется только на ноды с taint workload=heavy.
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  tolerations:
    - key: workload
      operator: Equal
      value: heavy
      effect: NoSchedule
  containers:
    - name: job
      image: batch:latest
```

### Часто используемые taints

| Taint (пример) | Назначение |
|----------------|------------|
| `node.kubernetes.io/not-ready:NoSchedule` | Обычно уже есть встроенный toleration у системных подов. |
| `node.kubernetes.io/unreachable:NoExecute` | После таймаута поды эвиктируются с недоступной ноды. |
| Собственный key, например `dedicated=system:NoSchedule` | Выделить ноды под системные компоненты; обычные поды туда не попадают. |

---

## Best practices и примеры из production

### Автомасштабирование

- **HPA:** задавать **minReplicas** не менее 2 для HA; **maxReplicas** — с запасом под пики. Указывать **requests** у контейнеров (без них HPA по CPU/memory не работает). Использовать **behavior** для сглаживания scale down.
- **VPA:** в production часто начинают с режима **Off** (рекомендации в статусе VPA), затем переходят на Recreate при наличии окна на перезапуск. Задавать **minAllowed/maxAllowed**, чтобы избежать экстремальных значений.
- **Cluster Autoscaler:** не держать поды с жёсткой привязкой к одной ноде без необходимости; использовать PDB для контроля eviction при scale-in.
- **KEDA:** для очередей задавать **queueLength** и **minReplicaCount** так, чтобы не создавать избыточное число потребителей и не оставлять очередь без потребителей надолго.

### Устойчивость

- **Probes:** для сервисов с трафиком всегда задавать **readinessProbe**; **livenessProbe** — по возможности без ложных срабатываний. Для долго стартующих приложений — **startupProbe**, чтобы liveness не убивал контейнер при инициализации.
- **PDB:** для любого Deployment/StatefulSet с несколькими репликами задавать PDB (**minAvailable** или **maxUnavailable**). Для single-replica БД — **maxUnavailable: 0** и понимание, что drain ноды заблокирован до ручного решения.
- **Anti-affinity:** для критичных приложений использовать **preferred** или **required** podAntiAffinity по **topology.kubernetes.io/zone** или **kubernetes.io/hostname**, чтобы реплики не оказались на одной ноде/в одной зоне.
- **Taints/Tolerations:** выделять ноды под специальные нагрузки через taints; не давать широкие tolerations обычным приложениям.

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **HPA + разумные min/max и behavior** | Предсказуемое масштабирование без «дрожания». |
| **PDB для всех stateful-сервисов** | Контроль добровольных eviction при обновлениях. |
| **Readiness + Liveness + Startup** | Корректное снятие с балансировки и перезапуск при сбоях. |
| **Anti-affinity по зоне/ноде** | Снижение риска потери всех реплик при отказе одной зоны/ноды. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **HPA без requests у контейнеров** | Метрики CPU/memory не работают для HPA. | Всегда задавать resources.requests. |
| **Нет PDB при нескольких репликах** | При drain все реплики могут уйти с ноды. | Задать minAvailable или maxUnavailable. |
| **Только liveness без readiness** | Трафик идёт на неготовые поды. | Добавить readinessProbe. |
| **Жёсткий required anti-affinity при малом числе реплик** | Поды могут остаться в Pending при нехватке нод. | Использовать preferred или увеличить число нод. |

---

## Дополнительные материалы

- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscaler/)
- [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [KEDA](https://keda.sh/)
- [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Assigning Pods to Nodes (Affinity, Taints/Tolerations)](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [3. Workloads и API-объекты](topic-3-workloads-api.md) — базовые probes и контроллеры.
