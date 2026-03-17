# 6. Хранилище в Kubernetes

Базовые объекты PV/PVC и StatefulSet описаны в [3. Workloads и API-объекты](topic-3-workloads-api.md); здесь углубление в жизненный цикл, CSI и эксплуатацию.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **PersistentVolume (PV)** | Ресурс хранилища в кластере (блочный том, NFS, облачный диск); задаётся администратором или provisioner'ом; имеет capacity, accessModes, reclaimPolicy. |
| **PersistentVolumeClaim (PVC)** | Запрос приложения на место из хранилища (размер, access mode, опционально storageClassName); при связывании с PV том монтируется в под. |
| **Binding (связывание)** | Сопоставление PVC с подходящим PV по capacity, accessModes и storageClass; выполняет контроллер volume binding или provisioner. |
| **StorageClass** | Класс хранилища и параметры для динамического выделения; задаёт provisioner и reclaimPolicy; при указании в PVC provisioner создаёт новый PV. |
| **Provisioner** | Компонент (часто CSI-драйвер), который по StorageClass создаёт том в бэкенде (облако, SAN) и создаёт объект PV. |
| **AccessMode (режим доступа)** | RWO (чтение/запись с одной ноды), ROX (чтение с многих нод), RWX (чтение/запись с многих нод); ограничение возможностей хранилища. |
| **ReclaimPolicy** | Поведение PV после удаления PVC: Retain (том сохраняется, PV в Released), Delete (том удаляется provisioner'ом). |
| **CSI (Container Storage Interface)** | Стандарт для подключения томов к подам; драйвер работает как поды в кластере и на нодах, создаёт и монтирует тома по запросу. |
| **VolumeSnapshot** | Снимок тома (CRD при использовании CSI); позволяет создать новый том из снимка или восстановить данные. |

---

## Введение: PV и PVC, связывание

- **PersistentVolume (PV)** — ресурс хранилища в кластере (том, NFS, облачный диск и т.д.). Задаётся администратором или создаётся provisioner’ом по StorageClass. Имеет capacity, accessModes, reclaimPolicy, тип (hostPath, nfs, csi и др.).
- **PersistentVolumeClaim (PVC)** — запрос на место: размер, access mode, опционально storageClassName. Контроллер volume binding связывает PVC с подходящим PV (по размеру, режиму доступа, классу). В поде в `volumes` указывают `persistentVolumeClaim.claimName`; kubelet монтирует том в контейнер.
- **Связывание:** при создании PVC контроллер ищет свободный PV с подходящими capacity и accessModes (и storageClass, если указан). При успехе PVC получает `status.phase: Bound` и привязку к конкретному PV. Если подходящего PV нет (статический сценарий) — PVC остаётся в Pending; при динамическом provisioner по StorageClass создаёт новый PV и привязывает его к PVC.

---

## AccessModes и ReclaimPolicy

**AccessModes:**

| Режим | Описание | Типичное использование |
|-------|----------|-------------------------|
| **ReadWriteOnce (RWO)** | Чтение/запись с одной ноды | Блочные тома (EBS, disk), один под на том. |
| **ReadOnlyMany (ROX)** | Чтение с многих нод | NFS, shared read-only данные. |
| **ReadWriteMany (RWX)** | Чтение/запись с многих нод | NFS, файловые хранилища (EFS, Azure Files). |

**ReclaimPolicy** (на PV или в StorageClass):

- **Retain** — после удаления PVC том не удаляется; данные сохраняются, PV переходит в Released. Администратор вручную чистит и при необходимости переиспользует.
- **Delete** — при удалении PVC provisioner удаляет и том (типично для облачных дисков при динамическом выделении).
- **Recycle** (устарел) — не использовать; предпочтительны Retain или Delete.

В production для критичных данных часто используют **Retain** или StorageClass с Retain, чтобы случайное удаление PVC не привело к потере тома.

---

## Примеры манифестов: статическое выделение

### PV и PVC (ручное создание PV, затем PVC)

```yaml
# Статическое выделение: администратор создаёт PV, пользователь — PVC.
# Binding произойдёт при совпадении capacity (PVC <= PV), accessModes и storageClass (или пустой класс).
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-app-data-01
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/app-01
  # В production вместо hostPath: csi, nfs или cloud volume (см. ниже).
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: default
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # После binding PVC будет привязан к pv-app-data-01 (до 5Gi из 10Gi).
```

### Под с монтированием PVC

```yaml
# Под использует том из PVC; том монтируется на одну ноду (RWO).
# При удалении пода данные остаются в PV; при пересоздании пода тот же PVC снова примонтируется.
apiVersion: v1
kind: Pod
metadata:
  name: app-with-pvc
  namespace: default
spec:
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: data
          mountPath: /app/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data-pvc
```

---

## StorageClass и динамическое выделение

**StorageClass** задаёт класс хранилища и параметры для **provisioner**’а (например, CSI). При создании PVC с указанием `storageClassName` provisioner создаёт новый PV и привязывает его к PVC (dynamic provisioning). Без storageClassName (или с `storageClassName: ""`) используются только статически созданные PV.

### StorageClass (пример: локальный provisioner / облачный диск)

```yaml
# StorageClass для динамического выделения. default: true — новые PVC без класса получат этот.
# provisioner зависит от окружения: e.g. kubernetes.io/aws-ebs, csi driver name и т.д.
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

- **volumeBindingMode: WaitForFirstConsumer** — PV создаётся только после того, как под, использующий PVC, будет запланирован (чтобы том создался в той же зоне/ноде, что и под). Рекомендуется для RWO в мультизонных кластерах.
- **allowVolumeExpansion: true** — разрешает расширение тома через редактирование `spec.resources.requests.storage` в PVC (поддерживается не всеми драйверами).

### PVC с StorageClass (динамическое выделение)

```yaml
# При создании этого PVC provisioner (например, EBS CSI) создаст новый том и привяжет его.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-dynamic-pvc
  namespace: default
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

---

## CSI (Container Storage Interface)

**CSI** — стандартный интерфейс для подключения внешних систем хранения. Компоненты: CSI driver (поды в кластере), sidecar-контейнеры (provisioner, attacher, resizer и т.д.). Драйвер регистрирует StorageClass’ы и обрабатывает create/delete/attach/mount томов. Динамическое выделение и снимки настраиваются через StorageClass и VolumeSnapshotClass.

### VolumeSnapshotClass и снимок (обзор)

```yaml
# VolumeSnapshotClass задаёт, как создавать снимки томов (зависит от CSI driver).
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
# VolumeSnapshot — запрос снимка существующего PVC. После создания снимка можно
# восстановить новый PVC из него (через dataSource в PVC) или использовать для бэкапов.
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: app-data-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: app-dynamic-pvc
```

Поддержка снимков зависит от драйвера и версии Kubernetes; детали — в документации конкретного CSI driver.

---

## StatefulSet + PVC: сохранность данных при потере пода

StatefulSet даёт стабильные имена подов и при использовании **volumeClaimTemplates** — отдельный PVC на каждый под (pod-0 → pvc-0, pod-1 → pvc-1). При удалении или ресхедуле пода тот же PVC снова монтируется к поду с тем же порядковым индексом, данные сохраняются. Обязателен Headless Service для стабильного DNS.

### StatefulSet с volumeClaimTemplate и Headless Service (production-ориентированный фрагмент)

```yaml
# Headless Service для стабильных DNS-имён подов (postgres-0.postgres-headless.namespace.svc.cluster.local).
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "1Gi"
              cpu: "500m"
  # Каждый под (postgres-0, postgres-1, ...) получит свой PVC из этого шаблона.
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        storageClassName: fast-ssd
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
```

При потере пода (eviction, сбой ноды) контроллер пересоздаст под с тем же именем (postgres-0); kubelet примонтирует тот же PVC — данные на томе сохраняются.

---

## Best practices и рекомендации для production

### Выбор типа хранилища

- **Блочные тома (RWO):** БД, очереди, всё, что требует низкой латентности и одного писателя. В облаке — EBS, Azure Disk, GCP PD; в кластере — локальные тома или CSI.
- **Файловые (RWX):** общие данные для нескольких подов (NFS, EFS, Azure Files). Выше латентность, подходят для кэша, общих конфигов, не для горячей БД.
- Не использовать **hostPath** для production-данных (привязка к ноде, нет переносимости, риски безопасности).

### Производительность

- Выбирать класс хранилища под нагрузку (gp3/io2 для EBS, размер и IOPS). При необходимости увеличивать размер тома (allowVolumeExpansion) или мигрировать на больший класс.
- **volumeBindingMode: WaitForFirstConsumer** для RWO в мультизонных кластерах — том создаётся в зоне пода, избегаем лишних cross-zone attach.

### Безопасность

- Ограничивать доступ к PVC/PV по RBAC; не давать широкие права на удаление PVC в production.
- Для чувствительных данных — шифрование at rest (часто на уровне облачного тома или через параметры StorageClass).
- В подах с томами использовать SecurityContext (runAsNonRoot, fsGroup при необходимости для прав на файлы тома).

### Масштабирование и управление

- StatefulSet для stateful-приложений; не использовать Deployment с одним репликой и общим PVC для БД — при ресхедуле возможна потеря данных или двойной writer.
- Регулярно проверять использование места (метрики, алерты); при приближении к лимиту — расширение тома или миграция.

### Резервное копирование и восстановление

- Не полагаться только на ReclaimPolicy Retain: при удалении PV данные можно потерять. Регулярные бэкапы: снапшоты томов (VolumeSnapshot где поддерживается), дампы БД на объектное хранилище, инструменты (Velero для кластера и т.д.).
- Тестировать восстановление из снимка/бэкапа в staging.

### Мониторинг и алертинг

- Метрики: использование места на томах (например, через node-exporter или CSI metrics), статус PVC (Pending, Bound), количество PV. Алерты на Pending PVC, на заполнение тома (>85% и т.п.).

---

## Комплексный пример: production-ready PostgreSQL с PVC

Идея: один инстанс PostgreSQL в StatefulSet, том через StorageClass с Retain, лимиты ресурсов, SecurityContext, Headless Service. Полноценный production-кластер (репликация, автоматический failover) выходит за рамки одного манифеста — ниже базовая схема.

```yaml
# StorageClass с Retain — при удалении PVC том не удалится (данные сохраняются).
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgres-retain
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
parameters:
  type: gp3
  iops: "3000"
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 5
            periodSeconds: 5
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        storageClassName: postgres-retain
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 30Gi
```

---

## Типичные ошибки и как их избежать

| Ошибка | Последствие | Решение |
|--------|-------------|---------|
| Deployment + один PVC для БД | При ресхедуле два пода могут примонтировать один RWO том или данные теряются. | StatefulSet + volumeClaimTemplate. |
| ReclaimPolicy Delete для критичных данных | Случайное удаление PVC удаляет том и данные. | StorageClass/PV с Retain; бэкапы. |
| hostPath для данных в production | Привязка к ноде, нет переносимости, риски. | CSI / облачные тома, NFS. |
| Игнорировать Pending PVC | Под не стартует, том не создаётся. | Проверить StorageClass, provisioner, квоты, события (kubectl describe pvc). |
| Нет бэкапов | Потеря тома или сбой ноды — потеря данных. | Снапшоты, дампы БД, Velero. |

---

## Чеклист для production-развёртывания с хранилищем

- [ ] Выбран подходящий тип хранилища (RWO/RWX) и StorageClass.
- [ ] Для критичных данных: ReclaimPolicy Retain и/или регулярные бэкапы.
- [ ] StatefulSet для stateful-приложений; volumeClaimTemplates вместо одного общего PVC.
- [ ] volumeBindingMode: WaitForFirstConsumer для RWO в мультизоне.
- [ ] Ресурсы (requests/limits) и SecurityContext заданы для подов с томами.
- [ ] Мониторинг использования места и алерты на Pending PVC и заполнение тома.
- [ ] Проверена процедура восстановления из снимка/бэкапа.

---

## Дополнительные материалы

- [Kubernetes — Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [CSI Documentation](https://kubernetes-csi.github.io/docs/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [3. Workloads и API-объекты](topic-3-workloads-api.md) — базовые PV, PVC, StatefulSet.
