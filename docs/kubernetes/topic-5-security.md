# 5. Безопасность в Kubernetes

**Цель:** настраивать безопасность на уровне кластера и приложений: RBAC (Role, ClusterRole, RoleBinding, ServiceAccount), Pod Security (Pod Security Standards, SecurityContext), сетевая изоляция (NetworkPolicies, Zero Trust), работа с секретами (Kubernetes Secrets, External Secrets, Vault). В разделе — примеры манифестов с комментариями и best practices для production.

Базовые объекты RBAC и Secret описаны в [3. Workloads и API-объекты](topic-3-workloads-api.md); здесь акцент на паттернах безопасности и ограничении доступа.

---

## RBAC

**RBAC** — авторизация по ролям: что разрешено делать с какими ресурсами. **Role** / **ClusterRole** задают правила (apiGroups, resources, verbs); **RoleBinding** / **ClusterRoleBinding** привязывают роль к User, Group или **ServiceAccount**. Role + RoleBinding действуют в рамках namespace; ClusterRole + ClusterRoleBinding — на весь кластер. Для приложений в подах создают отдельный ServiceAccount и выдают ему минимально необходимые права.

---

## Примеры манифестов: RBAC

### Role: ограничение доступа в рамках namespace

```yaml
# Роль разрешает только чтение Pod и Deployment в namespace.
# Подходит для разработчиков/операторов одного проекта (namespace).
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
```

### RoleBinding: привязка роли к пользователю или группе

```yaml
# Выдаёт роль app-reader пользователю или группе в этом namespace.
# subject.subjects[].name — имя пользователя/группы из аутентификации (e.g. OIDC, cert).
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: production
subjects:
  - kind: User
    name: developer@example.com
    apiGroup: rbac.authorization.k8s.io
  # - kind: Group
  #   name: developers
  #   apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount и RoleBinding для пода (доступ к API из приложения)

```yaml
# ServiceAccount для приложения в namespace. Под использует spec.serviceAccountName.
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-backend-sa
  namespace: production
---
# Роль: только чтение ConfigMap и Secret в своём namespace (например, для конфига).
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-backend-config-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    resourceNames: ["app-config", "app-db-secret"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-backend-config-reader-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: app-backend-sa
    namespace: production
roleRef:
  kind: Role
  name: app-backend-config-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole и ClusterRoleBinding (ограниченный доступ на весь кластер)

```yaml
# ClusterRole: только список нод и namespaces (для мониторинга/дашбордов).
# Не даёт изменять объекты — только get/list/watch.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/metrics", "namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding привязывает cluster-viewer к группе (например, из OIDC).
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-viewer-binding
subjects:
  - kind: Group
    name: view-only-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-viewer
  apiGroup: rbac.authorization.k8s.io
```

---

## Pod Security: Pod Security Standards и SecurityContext

**Pod Security Standards (PSS)** — встроенные уровни политики: **privileged**, **baseline**, **restricted**. При включении (admission) они запрещают или предупреждают о небезопасных настройках (например, hostPID, runAsRoot). Задаются через labels на namespace или через Pod Security Admission.

**SecurityContext** — на уровне пода и контейнера: от какого пользователя запускать процесс, группы для файлов, только для чтения корень ФС и т.д. Рекомендуется в production: **runAsNonRoot**, **runAsUser** (не 0), **readOnlyRootFilesystem** где возможно, **allowPrivilegeEscalation: false**.

---

## Примеры манифестов: Pod Security

### Namespace с Pod Security Standard (restricted)

```yaml
# Включение PSS "restricted" на namespace: только поды, соответствующие restricted, допускаются.
# Уровень enforced — нарушающие поды будут отклонены при создании.
apiVersion: v1
kind: Namespace
metadata:
  name: production-apps
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Pod с SecurityContext (production-ориентированный)

```yaml
# Безопасный под: не root, только чтение корня ФС, без повышения привилегий.
# Том для записи — отдельный (emptyDir или PVC).
apiVersion: v1
kind: Pod
metadata:
  name: app-secure
  namespace: production-apps
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 3000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:latest
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
```

### Кратко: ключевые поля SecurityContext

| Поле (pod/container) | Назначение |
|----------------------|------------|
| `runAsNonRoot` | Запрет запуска от root (uid 0). |
| `runAsUser` / `runAsGroup` | UID/GID процесса в контейнере. |
| `fsGroup` | GID для владения смонтированными томами (например, PVC). |
| `readOnlyRootFilesystem` | Корень ФС только для чтения; запись — в смонтированные тома. |
| `allowPrivilegeEscalation: false` | Запрет повышения привилегий. |
| `capabilities.drop: ["ALL"]` | Сброс Linux capabilities. |
| `seccompProfile.type: RuntimeDefault` | Профиль seccomp по умолчанию. |

---

## Network Security: NetworkPolicies и Zero Trust

**NetworkPolicy** — правила входящего (ingress) и исходящего (egress) трафика для подов (по selector). Требуется CNI с поддержкой NetworkPolicy (Calico, Cilium и др.). По умолчанию в большинстве установок трафик не ограничен; политика **deny all** закрывает весь трафик, затем разрешаются только нужные направления (Zero Trust: явное разрешение).

---

## Примеры манифестов: NetworkPolicy

### Deny all ingress и egress в namespace

```yaml
# Базовый шаг Zero Trust: запретить весь трафик по умолчанию.
# Все поды в namespace попадают под политику (podSelector: {}).
# Дальше добавляют политики "allow" только для нужных источников/назначений.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  # ingress: [] — не разрешено ничего
  # egress: []  — не разрешено ничего (осторожно: может сломать DNS и т.д.)
```

### Deny all ingress, разрешить egress (DNS и нужные цели)

```yaml
# Входящий трафик в namespace запрещён; исходящий — только DNS и нужные сервисы.
# Подходит как база: затем добавить allow ingress только для Ingress/контроллеров.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress-allow-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: TCP
          port: 443
```

### Разрешить ingress только от Ingress Controller и внутри namespace

```yaml
# Разрешаем входящий трафик к подам app только:
# 1) с подов ingress-контроллера (по label);
# 2) с других подов в том же namespace (внутрисервисное общение).
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-and-internal
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 80
```

### Разрешить трафик между подами по меткам

```yaml
# Backend принимает трафик только от подов с label component=frontend в том же namespace.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-from-frontend-only
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              component: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## Secrets

**Kubernetes Secret** — хранение чувствительных данных (base64 в etcd; в production желательно шифрование at-rest). Под монтирует Secret как файл или переменные окружения. Подробнее и примеры — в [3. Workloads и API-объекты](topic-3-workloads-api.md).

В production часто используют **External Secrets** или **Vault integration**: секреты хранятся во внешнем хранилище (HashiCorp Vault, AWS Secrets Manager и т.д.), оператор синхронизирует их в Kubernetes Secret. Так снижается риск утечки и централизуется ротация. Установка и настройка зависят от провайдера (Helm chart external-secrets, Vault Agent Injector и др.).

---

## Best practices и примеры из production

### RBAC

- **Минимальные права:** выдавать только нужные verbs и resources (get вместо create/delete, где достаточно чтения). Для приложений — отдельный ServiceAccount и Role с **resourceNames** там, где применимо.
- **Ограничивать по namespace:** по возможности использовать Role + RoleBinding, а не ClusterRole + ClusterRoleBinding.
- **Не использовать default ServiceAccount** для продакшн-подов: создавать именованный ServiceAccount и привязывать к нему роль.

### Pod Security

- Включать **Pod Security Standards** на namespace (например, **restricted** для production) и исправлять workload’ы под стандарт.
- В манифестах задавать **runAsNonRoot**, **runAsUser** (не 0), **readOnlyRootFilesystem**, **allowPrivilegeEscalation: false**, **capabilities.drop: ["ALL"]**. Исключения — только там, где реально нужно (например, init-контейнеры с монтированием hostPath).
- Для томов с записью использовать отдельные **emptyDir** или **PVC**, не полагаться на запись в корень образа.

### NetworkPolicy

- Начинать с **default deny all** в namespace, затем добавлять явные **allow** правила (Zero Trust). Учитывать egress для DNS (UDP 53) и при необходимости для метрик/логов.
- Разрешать ingress только от известных источников: Ingress Controller (по namespace/label), соседние поды по **podSelector**.
- Тестировать политики в dev/stage перед production; при сбоях проверять, не блокирует ли egress доступ к API/DNS.

### Secrets

- Не хранить секреты в образах и не логировать их. Использовать **Secret** и монтировать как файлы (предпочтительнее env для аудита).
- В production: **encryption at rest** для etcd и/или **External Secrets / Vault** для хранения и ротации.

---

## Практика: ограничить доступ к namespace и deny all NetworkPolicy

1. **Ограничить доступ к namespace:** создать Role с минимальными правами (например, только get/list/watch для pods, deployments в namespace `production`), RoleBinding привязать к пользователю или группе. Убедиться, что этот субъект не имеет ClusterRole с широкими правами.
2. **NetworkPolicy "deny all":** в выбранном namespace создать NetworkPolicy с `podSelector: {}` и `policyTypes: [Ingress, Egress]`, без правил ingress/egress (полный deny). Проверить: поды в этом namespace перестают принимать и отправлять трафик. Затем добавить egress для DNS (UDP 53) и при необходимости ingress от Ingress Controller — проверить доступ к приложению снаружи.

Проверка после deny all: из пода в этом namespace `curl` к другому сервису и извне к сервису не должны проходить до добавления разрешающих правил.

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Отдельный ServiceAccount на приложение** | Минимальные права через Role + RoleBinding. |
| **PSS restricted на production namespace** | Единый стандарт безопасности подов. |
| **SecurityContext без root** | runAsNonRoot, runAsUser, readOnlyRootFilesystem. |
| **NetworkPolicy deny all + явные allow** | Zero Trust: только разрешённый трафик. |
| **Секреты из Vault/External Secrets** | Централизованное хранение и ротация. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Широкий ClusterRole для всех** | Утечка прав при компрометации. | Role в namespace, минимальные verbs/resources. |
| **Запуск подов от root** | Усиление последствий взлома. | runAsNonRoot, runAsUser. |
| **Нет NetworkPolicy** | Любой под может стучаться в любой. | Deny all + явные allow по необходимости. |
| **Секреты в ConfigMap или в коде** | Риск утечки. | Secret или External Secrets. |

---

## Дополнительные материалы

- [Kubernetes — RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [External Secrets Operator](https://external-secrets.io/)
- [HashiCorp Vault — Kubernetes](https://developer.hashicorp.com/vault/docs/platform/k8s)
