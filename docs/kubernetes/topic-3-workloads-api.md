# 3. Workloads и API-объекты

**Цель:** уверенно работать с основными объектами Kubernetes: Pod (жизненный цикл, probes, init/sidecar), ConfigMap и Secret, типы Service, PersistentVolume/PVC, квоты и лимиты, QoS, RBAC, контроллеры (ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob). Понимать, как scheduler учитывает ресурсы и как устроены rolling update и StatefulSet.

---

## Pods

### Введение в Pod

**Pod** — минимальная единица развёртывания в Kubernetes: один или несколько контейнеров, разделяющих сеть (один IP), storage и namespace. Обычно в поде один основной контейнер; при необходимости добавляют sidecar или init-контейнеры. Под получает IP из подсети CNI и живёт на одной ноде; при падении ноды или удалении пода он не «переезжает» — создаётся новый под с новым IP.

### Жизненный цикл Pod (Pod Lifecycle)

Фазы: **Pending** (принят, контейнеры ещё не запущены или не все), **Running** (хотя бы один контейнер запущен), **Succeeded** / **Failed** (все контейнеры завершились успешно или с ошибкой). Переход в Succeeded/Failed возможен для подов с restartPolicy Never или OnFailure после завершения всех контейнеров. Удаление пода переводит его в **Terminating** до завершения graceful shutdown и удаления с ноды.

### Lifecycle hooks (хуки жизненного цикла)

**Lifecycle hooks** — действия, выполняемые kubelet в ключевые моменты жизни контейнера:

- **postStart** — выполняется сразу после создания контейнера (параллельно с основным процессом ENTRYPOINT/CMD). Типы: `exec` (команда в контейнере), `httpGet` (HTTP-запрос к localhost). Используется для инициализации после старта (например, регистрация в сервисе обнаружения). Если hook завершается с ошибкой, контейнер убивается и перезапускается согласно restartPolicy.
- **preStop** — выполняется **перед** отправкой SIGTERM контейнеру (при остановке пода: удаление, eviction, перезапуск). Типы: `exec`, `httpGet`. Kubelet ждёт завершения preStop в пределах **terminationGracePeriodSeconds** (по умолчанию 30 с), затем шлёт SIGTERM. Используется для корректного завершения: снятие с балансировщика, отмена регистрации, сохранение состояния. После preStop контейнер получает SIGTERM и может обработать его в приложении; по истечении grace period — SIGKILL.

Важно: preStop не блокирует удаление пода бесконечно — только в рамках grace period. Для длительных операций при остановке увеличивают `terminationGracePeriodSeconds` у пода.

### Probes (проверки состояния)

- **livenessProbe** — жив ли контейнер. При неуспехе kubelet перезапускает контейнер.
- **readinessProbe** — готов ли контейнер принимать трафик. При неуспехе под исключается из Endpoints Service (на него не идёт трафик).
- **startupProbe** — для медленно стартующих контейнеров: пока не успешна, liveness и readiness не проверяются.

Типы проверки: HTTP (путь и порт), TCP (порт открыт), exec (команда в контейнере). Параметры: initialDelaySeconds, periodSeconds, timeoutSeconds, failureThreshold.

### Init Containers

**Init-контейнеры** запускаются по очереди до основных контейнеров пода. Каждый должен завершиться успешно (exit 0). Используются для подготовки окружения (миграции БД, ожидание зависимости, копирование файлов). Основные контейнеры стартуют только после успешного завершения всех init.

### Sidecar Pattern

**Sidecar** — дополнительный контейнер в том же поде, который «сопровождает» основной: логирование, прокси, синхронизация. Контейнеры делят сеть (localhost) и при необходимости общие тома. Управление жизненным циклом sidecar и основного контейнера общее (под перезапускается целиком).

### Pod vs Container

Под — абстракция Kubernetes (один или несколько контейнеров, общие ресурсы). Контейнер — единица запуска внутри пода (один образ, свои limits/requests). Один под = один учёт в scheduler и одна «единица» масштабирования для контроллеров (Deployment масштабирует поды, а не контейнеры внутри них).

### Ephemeral Containers

**Ephemeral containers** — временные контейнеры, добавляемые в уже работающий под для отладки (например, shell). Не перезапускаются, не входят в spec по умолчанию. Использование: `kubectl debug` с опцией добавления ephemeral container.

---

## ConfigMap и Secret

- **ConfigMap** — объект с данными в виде ключ-значение или файлов. Монтируется в под как переменные окружения или как файлы в volume. Для несекретных конфигов (конфигурация приложения, пути).
- **Secret** — то же по структуре, но хранится в base64 (в etcd при включённом шифровании — зашифрован). Для паролей, TLS-сертификатов, токенов. Монтируется в под так же (env или volume). Не хранить чувствительные данные в plain text в ConfigMap.

---

## Service (типы)

**Service** — стабильная точка доступа к набору подов (по селектору labels). Типы:

| Тип | Назначение |
|-----|------------|
| **ClusterIP** | Виртуальный IP внутри кластера (по умолчанию). Доступ только из кластера. |
| **NodePort** | Порт (30000–32767) на каждой ноде; трафик с узла перенаправляется в ClusterIP и далее в под. |
| **LoadBalancer** | Обычно расширение NodePort: облачный контроллер создаёт внешний балансировщик и выдаёт внешний IP. |
| **Headless** | clusterIP: None — нет виртуального IP; DNS возвращает список IP подов (для StatefulSet и discovery). |
| **ExternalName** | CNAME на внешнее DNS-имя; для доступа к внешнему сервису по имени внутри кластера. |

kube-proxy (или ipvs) на каждой ноде реализует правила для ClusterIP/NodePort; трафик к ClusterIP:port уходит на один из эндпоинтов (под).

---

## PersistentVolume (PV) и PersistentVolumeClaim (PVC)

- **PersistentVolume (PV)** — ресурс хранилища в кластере (том, созданный администратором или provisioner’ом). Имеет capacity, access modes (RWO, ROX, RWX), storage class.
- **PersistentVolumeClaim (PVC)** — запрос пользователя на место из хранилища. Указывает size, access mode, опционально storageClassName. Контроллер (или provisioner) связывает PVC с PV (binding); в поде volume указывает на PVC, и том монтируется в контейнер.

Динамическое выделение: StorageClass + provisioner создают PV по запросу PVC. StatefulSet часто использует PVC с именем по шаблону для каждого пода.

---

## Resource Quotas и Limit Ranges

- **ResourceQuota** — ограничения по namespace: суммарные limits/requests по CPU и памяти, число объектов (pods, services, PVC и т.д.). Не даёт «съесть» весь кластер одним namespace.
- **LimitRange** — ограничения по умолчанию и мин/макс для подов и контейнеров в namespace. Можно задать default request/limit, чтобы каждый под получал лимиты, даже если не указаны в манифесте.

---

## Quality of Service (QoS)

**QoS (Quality of Service)** — класс пода, который Kubernetes вычисляет по **requests** и **limits** контейнеров (только основных; init-контейнеры в расчёт не входят). Класс определяет **порядок вытеснения (eviction)** при нехватке ресурсов на ноде: kubelet и ядро (OOM killer) вытесняют сначала поды с низким приоритетом.

### Правила определения класса

| Класс | Условие |
|-------|---------|
| **Guaranteed** | У **всех** контейнеров пода заданы и **limits**, и **requests** по CPU и памяти, причём **limits = requests** для каждого ресурса. |
| **Burstable** | Хотя бы у одного контейнера заданы requests или limits, но условие Guaranteed не выполнено (например, limits ≠ requests или задано только что-то одно). |
| **BestEffort** | Ни у одного контейнера нет ни requests, ни limits по CPU и памяти. |

### Порядок вытеснения при нехватке памяти

При давлении на память на ноде (eviction threshold) kubelet вытесняет поды в порядке: **сначала BestEffort**, затем **Burstable** (сначала те, чьё потребление больше всего превышает requests), в последнюю очередь **Guaranteed**. При OOM killer ядро использует **oom_score_adj** (kubelet выставляет его по QoS): у BestEffort он выше (вытесняются первыми), у Guaranteed — ниже.

Для **Burstable** при равном потреблении выше приоритет вытеснения у подов с меньшей разницей между usage и requests (сначала вытесняют тех, кто сильнее «переедает» относительно запрошенного).

### Примеры: какой под к какому классу относится

```yaml
# Guaranteed: у единственного контейнера limits = requests по CPU и памяти.
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "100m"
```

```yaml
# Burstable: limits заданы, но не равны requests (или заданы только requests).
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "256Mi"
          cpu: "200m"
```

```yaml
# BestEffort: нет ни requests, ни limits.
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
    - name: app
      image: nginx:1.25
```

Проверить класс: `kubectl get pod <name> -o jsonpath='{.status.qosClass}'`.

### Зачем это нужно

- **Критичные приложения** (БД, очереди) лучше задавать с **Guaranteed** (requests = limits), чтобы их вытесняли в последнюю очередь.
- **BestEffort** поды удобны для некритичных задач (batch, тесты), но при нехватке памяти они уйдут первыми.
- **Burstable** — компромисс: гарантированный минимум (requests) и возможность «переедать» до limits; при давлении на ноду порядок вытеснения зависит от того, насколько потребление превышает requests.

При давлении на ресурсы (OOM, eviction по памяти) scheduler не планирует новые поды на переполненную ноду; kubelet вытесняет уже запущенные поды по описанному порядку.

### Работа Scheduler с ресурсами

Scheduler при выборе ноды смотрит на **requests** подов (уже запущенных и планируемого): нода должна иметь достаточный allocatable. **Limits** ограничивают контейнер через cgroups на ноде; scheduler по limits не фильтрует ноды (используются requests). Поэтому задавать requests обязательно для предсказуемого размещения и учёта квот.

---

## RBAC (Role-Based Access Control)

**RBAC** — авторизация в Kubernetes на основе ролей. Объекты: **Role** / **ClusterRole** (набор правил: какие API-группы, ресурсы и глаголы разрешены), **RoleBinding** / **ClusterRoleBinding** (привязка роли к субъекту: User, Group, ServiceAccount). Субъект с RoleBinding получает права в рамках namespace; ClusterRoleBinding — в рамках всего кластера. Для доступа подов к API (например, приложениям в кластере) создают ServiceAccount и выдают ему нужную роль через RoleBinding/ClusterRoleBinding.

---

## Controllers

### Введение

**Контроллер** — цикл reconciliation: смотрит на желаемое состояние (например, replicas: 3 у Deployment) и текущее (число подов), создаёт или удаляет объекты, чтобы привести текущее к желаемому. Контроллеры встроены в kube-controller-manager и в других компонентах (например, DaemonSet контроллер).

### ReplicaSet

**ReplicaSet** — поддерживает заданное число реплик подов по селектору. Создаёт/удаляет поды по шаблону (podTemplate). Обычно не создают ReplicaSet вручную — им управляет **Deployment**.

### Deployment

**Deployment** — управляет ReplicaSet и обеспечивает обновление подов (rolling update, recreate). Поля **strategy**, **maxSurge**, **maxUnavailable** задают, как обновлять: сколько новых подов создавать сверх желаемого числа и сколько старых может быть недоступно во время обновления. Обновление: смена podTemplate (образ, env) в spec; Deployment создаёт новый ReplicaSet и потихоньку переводит поды на него.

### Rolling Update: maxSurge и maxUnavailable

- **maxSurge** — сколько подов сверх desired может существовать во время обновления (число или процент). Например, 1 или 25% — можно временно иметь на один под больше.
- **maxUnavailable** — сколько подов может быть недоступно (число или процент). 0 — нельзя уменьшать доступное число (обновление только за счёт surge).

Комбинация задаёт скорость и «безопасность» обновления: при maxUnavailable=0 и maxSurge=1 сначала поднимается новый под, потом гасится старый.

### StatefulSet

**StatefulSet** — workload для stateful-приложений: поды имеют стабильные идентификаторы (pod-name-0, pod-name-1, …), стабильное сетевое имя (через Headless Service) и при необходимости свой том (volumeClaimTemplates). Запуск и обновление по умолчанию **упорядочены** (0, затем 1, затем 2…). **Headless Service** (clusterIP: None) нужен для DNS-имён подов (pod-name-0.headless-svc.namespace.svc.cluster.local).

### DaemonSet

**DaemonSet** — по одному поду на каждой ноде (или на нодах по селектору). При добавлении ноды под создаётся автоматически. Используется для агентов (логирование, мониторинг, CNI, kube-proxy в некоторых установках).

### Job

**Job** — один или несколько подов до успешного завершения (exit 0). Под перезапускается при сбое до backoffLimit. Подходит для разовых задач (миграции, расчёт).

### CronJob

**CronJob** — по расписанию (cron-выражение) создаёт Job. Каждый запуск — новая Job. Параметры: schedule, concurrencyPolicy (Allow, Forbid, Replace), successfulJobsHistoryLimit, failedJobsHistoryLimit.

---

## Примеры манифестов

Ниже — примеры манифестов для всех сущностей, описанных в разделе. Подставьте свой namespace или используйте `default`.

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    env: production
```

### Pod с probes и init-контейнером

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
  labels:
    app: myapp
spec:
  initContainers:
    - name: wait-deps
      image: busybox:1.36
      command: ["sh", "-c", "echo ready && exit 0"]
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "50m"
        limits:
          memory: "128Mi"
          cpu: "200m"
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3
```

### Pod с sidecar (два контейнера в одном поде)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
  labels:
    app: myapp
spec:
  containers:
    - name: app
      image: myapp:1
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /var/log/app/app.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
  volumes:
    - name: shared-logs
      emptyDir: {}
```

### Pod с lifecycle hooks (postStart и preStop)

```yaml
# postStart — после старта контейнера (например, уведомить сервис обнаружения).
# preStop — перед SIGTERM: снять под с приёма трафика, дать время завершить запросы.
# terminationGracePeriodSeconds — время на выполнение preStop и обработку SIGTERM.
apiVersion: v1
kind: Pod
metadata:
  name: app-with-lifecycle-hooks
  labels:
    app: myapp
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: app
      image: myapp:1
      ports:
        - containerPort: 8080
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo started >> /tmp/lifecycle.log"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5; echo stopping >> /tmp/lifecycle.log"]
      # Альтернатива preStop через HTTP (если приложение отдаёт endpoint для drain):
      # lifecycle:
      #   preStop:
      #     httpGet:
      #       path: /prestop
      #       port: 8080
```

### ConfigMap и Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  config.json: |
    {"port": 8080}
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  password: "my-secret-password"
```

Использование в поде (env и volume):

```yaml
spec:
  containers:
    - name: app
      image: myapp:1
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: password
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

### Service: ClusterIP и Headless

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
spec:
  clusterIP: None
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
```

### Service: LoadBalancer (облако создаёт внешний балансировщик)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
  # loadBalancerIP: "1.2.3.4"   # опционально, в облаках с поддержкой
```

### Service: ExternalName (CNAME на внешний DNS)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

Внутри кластера обращение к `external-db.default.svc.cluster.local` будет резолвиться в `db.example.com`.

### NodePort (доступ снаружи на порт ноды)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

### PersistentVolume (PV) и PersistentVolumeClaim (PVC)

Ручное выделение: администратор создаёт PV, пользователь — PVC; при совпадении capacity и accessMode происходит binding.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual-1
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/pv1
  # или cloud block volume, nfs, csi и т.д.
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # storageClassName: ""   # пустая строка — только статический PV без класса
```

### Под с томом (PVC)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-pvc
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

### ReplicaSet (обычно создаётся Deployment’ом)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
```

### Deployment с rolling update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

### StatefulSet с Headless Service и volumeClaimTemplate

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector:
    app: db
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 2
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
```

### DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
        - name: collector
          image: myorg/log-collector:1
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

### Job и CronJob

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-db
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: myapp:migrate
          command: ["/bin/migrate.sh"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: myapp:backup
```

### ResourceQuota и LimitRange

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "10"
    persistentvolumeclaims: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: ns-limits
spec:
  limits:
    - type: Container
      default:
        cpu: "100m"
        memory: "128Mi"
      defaultRequest:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "2"
        memory: 2Gi
```

### RBAC: Role и RoleBinding для ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["configmaps", "pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: default
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
```

В поде указать `serviceAccountName: app-sa`, чтобы процесс в контейнере использовал этот ServiceAccount при обращении к API кластера.

### ClusterRole и ClusterRoleBinding (права на весь кластер)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-nodes
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes/metrics"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-for-app
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: read-nodes
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount (отдельно, для использования в Pod)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
# автоматически создаётся secret для imagePullSecrets и API-токена (если нужен)
```

В spec пода: `serviceAccountName: app-sa`.

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Задавать requests/limits** | Scheduler и квоты работают с requests; limits защищают ноду от перегрузки. |
| **Readiness + liveness** | Readiness — не слать трафик, пока не готов; liveness — перезапуск при зависании. |
| **preStop для graceful shutdown** | Перед SIGTERM снять под с приёма трафика (drain), дать завершить запросы; при необходимости увеличить terminationGracePeriodSeconds. |
| **Guaranteed для критичных workload'ов** | При нехватке памяти на ноде такие поды вытесняются последними (QoS). |
| **ConfigMap/Secret в volume** | При изменении ConfigMap/Secret можно перезапускать под или использовать подводку через kubelet (зависит от версии и настроек). |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Без readiness** | Трафик идёт на под до готовности приложения — ошибки и таймауты. | Всегда readinessProbe для сервисов с трафиком. |
| **Игнорировать preStop при активном трафике** | При остановке пода новые запросы могут прийти до исключения из Service; обрыв соединений. | Использовать preStop (sleep или вызов endpoint drain) и адекватный terminationGracePeriodSeconds. |
| **BestEffort для критичных приложений** | При нехватке памяти на ноде такие поды вытесняются первыми (QoS). | Задавать requests и limits; для максимальной защиты от eviction — Guaranteed (limits = requests). |
| **Deployment для stateful** | Поды получают случайные имена и тома; при ресхедуле теряется связь с данными. | StatefulSet + PVC для БД и подобного. |

---

## Дополнительные материалы

- [Kubernetes — Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes — Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes — StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes — RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Configure Quality of Service](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
