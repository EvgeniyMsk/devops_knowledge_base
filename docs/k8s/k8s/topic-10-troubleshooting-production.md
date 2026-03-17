# 10. Отладка и эксплуатация в production (Troubleshooting & Production)

---
Типичные проблемы (Pending Pods, ImagePullBackOff, OOMKilled, NetworkPolicy, latency etcd), инструменты (kubectl debug, stern, k9s, kubectl top, tcpdump) и сценарии (падение ноды, недоступность API server, массовый рестарт). Примеры и best practices для production.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Pending (под)** | Фаза пода: принят API, но ещё не запланирован на ноду или не запущены контейнеры; при застревании — смотреть Events (ресурсы, nodeSelector, taints, PVC). |
| **ImagePullBackOff / ErrImagePull** | Состояние контейнера: образ не удаётся скачать (неверное имя/тег, нет доступа к registry, сеть); kubelet повторяет pull с backoff. |
| **OOMKilled** | Причина завершения контейнера: потребление памяти превысило limit; ядро (OOM killer) завершило процесс. |
| **CrashLoopBackOff** | Под постоянно перезапускается (контейнер падает); backoff — задержка между рестартами увеличивается. |
| **Eviction** | Вытеснение пода с ноды при нехватке ресурсов (память, диск) или при drain ноды; порядок — по QoS и приоритету. |
| **imagePullSecrets** | Ссылка на Secret с учётными данными для доступа к приватному container registry; kubelet использует при pull образа. |
| **kubectl debug** | Возможность подключиться к поду для отладки: ephemeral container или копия пода с изменённым command; не требует перезапуска приложения. |
| **Readiness / Liveness probe** | Проверки состояния контейнера: при неуспехе readiness под исключается из Service; при неуспехе liveness контейнер перезапускается. |

---

## Частые проблемы: диагностика и решение

### Pending Pods

**Симптом:** под остаётся в состоянии `Pending`; не запланирован на ноду.

**Причины и проверки:**

| Причина | Как проверить | Решение |
|--------|----------------|---------|
| Недостаточно ресурсов на нодах | `kubectl describe pod <name>` — в Events: "0/X nodes are available: insufficient cpu/memory" | Увеличить ресурсы нод, уменьшить requests подов или добавить ноды (Cluster Autoscaler). |
| Не подходит nodeSelector / affinity / taints | Events: "0/X nodes match node selector" или "node(s) had taint" | Добавить подходящие ноды, убрать/ослабить affinity или добавить tolerations. |
| PVC не привязан (Pending) | Events: "waiting for volume to be created" | Проверить StorageClass, provisioner; `kubectl get pvc` — статус Bound. |
| Нет подходящих нод (все NotReady) | `kubectl get nodes` | Восстановить ноды, проверить kubelet и сеть. |

```bash
# Быстрая диагностика
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl top nodes
```

---

### ImagePullBackOff

**Симптом:** под в `ImagePullBackOff` или `ErrImagePull`; образ не удаётся скачать.

**Причины и проверки:**

| Причина | Как проверить | Решение |
|--------|----------------|---------|
| Неверное имя/тег образа | `kubectl describe pod` — Image: ..., сообщение об ошибке | Исправить image в манифесте/чарте. |
| Приватный registry без авторизации | Events: "unauthorized" / "pull access denied" | Создать Secret типа docker-registry и указать imagePullSecrets в поде. |
| Сеть / DNS до registry | Events: "connection timed out" | Проверить сеть с ноды, DNS, firewall, proxy. |
| Несуществующий тег или образ удалён | Events: "not found" | Использовать существующий тег или digest. |

**Пример: Secret для приватного registry и использование в поде**

```yaml
# imagePullSecrets нужен, чтобы kubelet на ноде мог аутентифицироваться в приватном registry.
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: default
type: kubernetes.io/dockerconfigjson
data:
  # Получить: kubectl create secret docker-registry regcred --docker-server=registry.example.com \
  #   --docker-username=... --docker-password=... --dry-run=client -o yaml
  .dockerconfigjson: <base64-encoded-docker-config>
---
apiVersion: v1
kind: Pod
metadata:
  name: app-private-image
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: registry.example.com/myorg/myapp:v1
```

```bash
# Проверка образа и событий
kubectl describe pod <pod-name> | grep -A5 Events
kubectl get events -n <ns> --field-selector involvedObject.name=<pod-name>
```

---

### OOMKilled

**Симптом:** контейнер перезапускается; в `kubectl describe pod` — `Last State: Terminated, Reason: OOMKilled`.

**Причина:** потребление памяти контейнером превысило `limits.memory` (или лимит ноды); ядро убивает процесс (OOM killer).

**Действия:**

1. Увеличить **limits.memory** и/или **requests.memory** (если приложению реально нужно больше).
2. Найти утечки памяти в приложении (профилирование, метрики).
3. Временно снять или ослабить limit только для отладки (в production не оставлять без limit — один под может занять всю ноду).

```yaml
# Пример: адекватные requests/limits и мониторинг; при OOMKilled — поднять limit и проверить приложение.
containers:
  - name: app
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

```bash
# Убедиться, что причина — OOM
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
kubectl top pod <pod> -n <ns>
```

---

### NetworkPolicy блокирует трафик

**Симптом:** поды не могут достучаться друг до друга или до внешних сервисов; логи "connection refused" / timeout при наличии сервиса.

**Проверки:**

1. В namespace включена политика "default deny" (ingress/egress пустые) — нужны явные allow-правила.
2. Egress для DNS (UDP 53) и при необходимости для API (HTTPS) и метрик.
3. Ingress разрешён только от нужных источников (Ingress Controller, соседние поды).

См. примеры и разбор в [5. Безопасность](topic-5-security.md). Быстрая проверка: временно удалить или ослабить NetworkPolicy и повторить запрос; если заработало — скорректировать правила.

```bash
# Список политик в namespace
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>
# Проверка с другого пода
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl -v http://<service>.<ns>.svc.cluster.local/
```

---

### etcd latency

**Симптом:** медленные ответы API (kubectl зависает), алерты на высокую латентность etcd; возможны таймауты при большом числе объектов или тяжёлой нагрузке на запись.

**Причины и действия:**

| Причина | Действие |
|--------|----------|
| Перегрузка диска (IO, место) | Проверить место и IO на нодах etcd; увеличить диск, использовать SSD. |
| Слишком большой размер кластера / много объектов | Удалять неиспользуемые объекты; разбивать на несколько кластеров. |
| Недостаточно ресурсов (CPU/RAM) у etcd | Выделить больше ресурсов под etcd. |
| Сетевые задержки между репликами | Размещать etcd в одной зоне/дата-центре с низкой латентностью. |

```bash
# Метрики etcd (если доступны)
# ETCDCTL_API=3 etcdctl endpoint status --endpoints=... --cert=...
# В managed-кластерах — мониторинг через провайдера (метрики control plane).
```

---

## Инструменты

### kubectl debug

Запуск временного контейнера в поде для отладки (ephemeral container) или копирование пода с изменённым образом (например, замена образа на busybox).

```bash
# Эфемерный контейнер в существующем поде (Kubernetes 1.23+)
kubectl debug -it <pod-name> -n <ns> --image=busybox --target=<container_name> -- sh

# Запуск копии пода с заменой образа (например, если основной образ без shell)
kubectl debug <pod-name> -n <ns> --copy-to=debug-pod --share-processes --container=<name> --image=nicolaka/netshoot -it -- sh
```

Удобно проверять сеть, DNS, процессы внутри namespace пода без пересоздания основного контейнера.

---

### stern

**stern** — агрегация логов по нескольким подам по селектору (подобно `kubectl logs` с -l, но с цветами и фильтрами).

```bash
# Логи всех подов с label app=myapp в namespace default
stern myapp -n default

# С таймстампом и фильтром по строке
stern myapp -n default -t --include "error"
```

Установка: см. [stern](https://github.com/stern/stern).

---

### k9s

**k9s** — TUI для кластера: навигация по ресурсам, логи, describe, перезапуск подов с клавиатуры. Запуск: `k9s` (выбор контекста и namespace через интерфейс). Полезно для быстрого обзора подов, нод и событий.

---

### kubectl top

Показывает использование CPU и памяти по нодам и подам (данные из metrics-server).

```bash
kubectl top nodes
kubectl top pods -n <namespace>
kubectl top pods -n <namespace> -l app=myapp
```

Если `kubectl top` не работает — проверить, установлен ли metrics-server и доступен ли он.

---

### tcpdump в Pod

Для отладки сети внутри кластера можно запустить под с привилегированным контейнером и tcpdump (или использовать `kubectl debug` с образом типа nicolaka/netshoot).

```yaml
# Временный под для захвата трафика в namespace (осторожно: привилегированный под).
# Удалить после отладки. В production предпочтительнее ограничить namespace и использовать только при необходимости.
apiVersion: v1
kind: Pod
metadata:
  name: tcpdump-debug
  namespace: default
spec:
  containers:
    - name: tcpdump
      image: nicolaka/netshoot
      command: ["tcpdump", "-i", "any", "-n", "host", "10.96.0.0/12"]
      securityContext:
        privileged: true
        capabilities:
          add: ["NET_RAW", "NET_ADMIN"]
  restartPolicy: Never
```

Альтернатива: `kubectl debug <pod> --image=nicolaka/netshoot -it -- tcpdump -i any -n`.

---

## Сценарии: как действовать

### Падение ноды (Node NotReady)

1. **Подтвердить:** `kubectl get nodes` — нода в статусе NotReady.
2. **Поды на ноде:** `kubectl get pods -A -o wide | grep <node-name>` — поды в Unknown или Terminating.
3. **Решения:**
   - Восстановить ноду (перезапуск, замена в облаке) — после возврата ноды kubelet подхватит поды или контроллеры пересоздадут их на других нодах.
   - При долгой недоступности: принудительно удалить под на мёртвой ноде: `kubectl delete pod <pod> -n <ns> --grace-period=0 --force` — контроллер (Deployment, StatefulSet) создаст новый под на живой ноде.
4. **StatefulSet с PVC:** под будет пересоздан с тем же именем и примонтирует тот же PVC; убедиться, что том доступен с другой ноды (RWO — том должен быть в той же зоне/ноде в облаке).
5. **Проверить** реплики приложений, PDB и алерты после восстановления.

---

### Недоступен API server

1. **Проверить доступ:** `kubectl get nodes` или любой запрос к API; таймаут или 5xx.
2. **Возможные причины:** падение/перезагрузка control plane, сеть до API, перегрузка etcd, проблемы с аутентификацией/балансировщиком перед API.
3. **Действия:**
   - В managed-кластере (EKS, GKE, AKS): проверить статус в консоли провайдера; инциденты control plane.
   - В своём кластере: проверить поды/ноды control plane (kube-apiserver, etcd), логи и метрики; восстановить кворум etcd при необходимости (см. [2. Архитектура](topic-2-architecture.md)).
4. **Поды и kubelet:** без API kubelet не получает новый список подов, но уже запущенные контейнеры продолжают работать; после восстановления API кластер придёт в согласованное состояние.

---

### Массовый рестарт подов

1. **Масштаб:** `kubectl get pods -A | grep -E 'Restarting|CrashLoopBackOff|Error'`; посмотреть события по namespace: `kubectl get events -A --sort-by='.lastTimestamp'`.
2. **Типичные причины:**
   - Новый деплой с ошибкой (неверный образ, конфиг, лимиты) — откат Deployment/релиза.
   - OOMKilled на многих подах — повысить memory limit или найти утечку.
   - Падение/обновление нод — поды переезжают на другие ноды; проверить ресурсы и PDB.
   - Проблема с зависимостью (БД, внешний API) — readiness не проходит, поды перезапускаются или не готовы; устранить зависимость.
3. **Действия:**
   - Откатить образ/конфиг: `kubectl rollout undo deployment/<name> -n <ns>`.
   - Временно увеличить ресурсы или исправить конфиг (ConfigMap/Secret) и перезапустить: `kubectl rollout restart deployment/<name> -n <ns>`.
   - Проверить лимиты квот namespace (ResourceQuota) и лимиты нод.
4. **Мониторинг:** алерты на CrashLoopBackOff и на долю неготовых подов (см. [7. Наблюдаемость](topic-7-observability.md)).

---

## Best practices для production

- **Всегда задавать requests/limits** — избегать OOM и перегрузки нод; мониторить `kubectl top` и метрики.
- **Readiness и liveness probes** — корректно настроенные, чтобы не перезапускать здоровые поды и не слать трафик на неготовые (см. [8. Автомасштабирование и отказоустойчивость](topic-8-autoscaling-resilience.md)).
- **PDB** для критичных приложений — контролировать добровольные eviction при обновлениях нод.
- **Мониторинг и алерты** — NodeNotReady, PodCrashLooping, нехватка места на дисках, латентность API/etcd; runbook на каждый алерт.
- **Документированные runbook’и** для сценариев: падение ноды, недоступность API, массовый рестарт; регулярные учения (game days).
- **Ограниченный доступ к production** — изменения через GitOps; прямой kubectl только для отладки и инцидентов с последующим приведением состояния в Git.

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Сначала describe/events** | Быстро сужают круг причин (Pending, ImagePullBackOff, OOM, сеть). |
| **Откат перед долгой отладкой** | Восстановить сервис откатом, затем разбирать причину. |
| **Отладка в отдельном namespace** | debug-поды и временные тесты не в production namespace. |
| **Фиксировать шаги при инциденте** | Post-mortem и обновление runbook. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Менять production «наугад»** | Риск усугубить сбой. | Следовать runbook, откатываться при неуверенности. |
| **Без limits на память** | Один под может положить ноду (OOM). | Всегда задавать limits; мониторить потребление. |
| **Игнорировать Events** | Теряется время на поиск причины. | Первым делом смотреть describe pod и events. |

---

## Дополнительные материалы

- [Kubernetes — Troubleshooting](https://kubernetes.io/docs/tasks/debug/)
- [kubectl debug (Ephemeral Containers)](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-running-pod/#ephemeral-container)
- [stern](https://github.com/stern/stern)
- [k9s](https://k9scli.io/)
- [2. Архитектура](topic-2-architecture.md) — control plane, etcd
- [5. Безопасность](topic-5-security.md) — NetworkPolicy
- [7. Наблюдаемость](topic-7-observability.md) — алерты и мониторинг
