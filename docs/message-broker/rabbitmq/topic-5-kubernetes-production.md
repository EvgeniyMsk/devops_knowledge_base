# 5. Kubernetes и Production

---
Тема про практическое развёртывание RabbitMQ в Kubernetes: **StatefulSet и persistent volumes**, Helm‑чарты, RabbitMQ Cluster Operator, health checks (liveness/readiness), ресурсы, обновления и базовые production‑настройки. Цель — уметь поддерживать RabbitMQ в проде, а не просто “поднять”.

---

## Базовые сущности в Kubernetes (в контексте RabbitMQ)

| Сущность | Зачем нужна |
|---------|-------------|
| **StatefulSet** | Стабильные имена подов (`rabbitmq-0`, `rabbitmq-1`), предсказуемое обновление, удобен для stateful‑нагрузок. |
| **PersistentVolume / PVC** | Данные RabbitMQ (mnesia, сообщения, метаданные) должны переживать рестарт/пересоздание пода. |
| **Headless Service** | Стабильные DNS‑имена подов для межнодовой коммуникации кластера. |
| **PodDisruptionBudget** | Защита от “случайного” одновременного сноса нескольких реплик при обслуживании нод. |
| **Readiness/Liveness probes** | Readiness — когда можно принимать трафик; Liveness — когда контейнер надо перезапустить. |

---

## Два подхода: Helm chart vs Operator

### Helm chart (быстро и достаточно часто)

Подходит, когда:

- нужен управляемый шаблон деплоя;
- вы готовы сами следить за жизненным циклом (апгрейды, тюнинг);
- нет требования к “Day‑2 operations” через CRD.

Часто используют `bitnami/rabbitmq` (или официальный chart, если он подходит под ваши требования).

### RabbitMQ Cluster Operator (production‑ориентированно)

Подходит, когда:

- нужен “правильный” lifecycle через CRD (`RabbitmqCluster`);
- важны автоматизация и эксплуатационные операции (обновления, конфигурация, TLS, plugins) в Kubernetes‑стиле.

---

## Практика: развёртывание через Helm (пример)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install rabbitmq bitnami/rabbitmq \
  --namespace rabbitmq --create-namespace \
  --set replicaCount=3 \
  --set auth.username=app \
  --set auth.password='change-me' \
  --set auth.erlangCookie='change-me-too' \
  --set persistence.enabled=true \
  --set persistence.size=20Gi \
  --set resources.requests.cpu=200m \
  --set resources.requests.memory=512Mi \
  --set resources.limits.cpu=1 \
  --set resources.limits.memory=1Gi
```

Комментарий:

- **replicaCount=3** — минимальная разумная база для HA (кворум).
- **persistence.enabled** обязательно: без диска кластер переживёт только “пока не перезапустили”.
- Requests/limits задают предсказуемость планирования и защиту от OOM.
- Пароли и cookie в production не передавать через CLI — использовать values‑файл + Secret/Vault.

---

## Production best practices: storage и безопасность данных

### Persistent storage

- Используйте **быстрый диск** (особенно для quorum очередей).
- Включайте **retention** и бэкапы там, где это применимо (в зависимости от стратегии хранения сообщений).
- Планируйте capacity: backlog сообщений = диск + память + latency.

### TLS и доступ

- Не открывайте Management UI наружу без TLS и auth.
- Разделяйте доступ на уровне **vhost/users/permissions**.
- Убирайте дефолтного `guest` (или запрещайте доступ не с localhost).

---

## Probes: liveness/readiness (как делать безопасно)

Правило: readiness должен становиться true **только когда нода готова принимать publish/consume** (подключения, синхронизация, лидерство).

Если readiness слишком “оптимистичный”:

- сервис начнёт слать трафик на ноду, которая не готова → ошибки и ретраи

Если liveness слишком “агрессивный”:

- вы будете перезапускать рабочую ноду при кратковременной нагрузке → хуже

Best practice: использовать готовые health endpoints (зависит от чарта/operator) и не “curl management API каждую секунду”.

---

## Ресурсы: limits, OOM и backpressure

- RabbitMQ чувствителен к **памяти**: при memory alarm включается flow control (backpressure), producers замедляются.
- В Kubernetes при слишком маленьком memory limit вы получите **OOMKilled** и flapping.

Практика:

- ставить реальные **requests**, чтобы scheduler не запланировал “слишком плотно”
- мониторить memory alarms и рост `messages_unacknowledged`
- тюнить prefetch у consumer’ов, а не “лечить” всё ресурсами

---

## Autoscaling consumer’ов (пример идеи)

RabbitMQ сам не масштабирует ваши consumer‑приложения — это делает Kubernetes.

Пример логики: HPA по пользовательской метрике “очередь растёт” (через Prometheus).

```yaml
# Концептуальный пример: HPA по метрике backlog (нужен prometheus-adapter).
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: tasks-consumer-hpa
  namespace: rabbitmq
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: tasks-consumer
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: rabbitmq_queue_messages_ready
        target:
          type: AverageValue
          averageValue: "1000"
```

Комментарий:

- Метрика должна быть “правильной” (например, backlog на очередь/consumer group).
- Масштабирование consumer’ов должно учитываться вместе с **prefetch** (иначе in‑flight взлетит).

---

## Rolling updates и PDB

Best practices:

- Обновлять ноды RabbitMQ **по одной** и проверять состояние кластера после каждой.
- Настроить **PodDisruptionBudget**, чтобы `kubectl drain` не убил сразу несколько реплик.

Пример PDB:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: rabbitmq-pdb
  namespace: rabbitmq
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: rabbitmq
```

---

## Небольшая эксплуатационная шпаргалка

```bash
# Поды и их состояние
kubectl get pods -n rabbitmq -o wide

# События (часто видно OOM/Probe failures)
kubectl get events -n rabbitmq --sort-by='.lastTimestamp'

# Логи
kubectl logs -n rabbitmq rabbitmq-0

# Проверка очередей (если есть доступ в pod)
kubectl exec -n rabbitmq -it rabbitmq-0 -- rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

---

## Дополнительные материалы

- [RabbitMQ Cluster Operator](https://www.rabbitmq.com/kubernetes/operator/operator-overview.html)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Bitnami RabbitMQ chart](https://github.com/bitnami/charts/tree/main/bitnami/rabbitmq)
