# 13. Docker и Kubernetes (связка)

**Цель темы:** понимать, где заканчивается зона ответственности Docker и начинается Kubernetes: как образ Docker используется в поде, как ENTRYPOINT/CMD соотносятся с полем command контейнера, зачем нужен imagePullPolicy и securityContext, и чем отличаются container runtime (containerd, CRI-O) от «классического» Docker daemon.

---

## Определения терминов

### Образ в Kubernetes

**Образ** в манифесте пода — это тот же OCI/Docker-образ (из registry по имени и тегу или по digest). Kubernetes не собирает образы; kubelet на ноде пуллит образ через **container runtime** (containerd, CRI-O и т.д.) и запускает контейнер по спецификации пода. Формат образа не меняется: образ, собранный через `docker build` и запушенный в registry, подходит для Kubernetes без пересборки.

### command и args в Pod

В спецификации контейнера в поде поля **command** и **args** переопределяют **ENTRYPOINT** и **CMD** образа. `command` соответствует ENTRYPOINT, `args` — аргументам (вместо CMD). Если задать только `args`, ENTRYPOINT образа остаётся, аргументы подставляются вместо CMD. Это важно для переопределения точки входа без пересборки образа (например, передать флаги приложению).

### imagePullPolicy

**imagePullPolicy** — когда kubelet качает образ с registry: **Always** (каждый раз проверять по тегу), **IfNotPresent** (пуллить, если образа нет локально), **Never** (только локальный образ). Для тега `latest` по умолчанию используется Always; для конкретного тега (например, v1.0.0) — IfNotPresent. Для воспроизводимости в production используют теги по версии или по digest и при необходимости явно задают IfNotPresent или Always.

### securityContext

**securityContext** — настройки безопасности контейнера (и пода): runAsUser, runAsNonRoot, readOnlyRootFilesystem, capabilities, seccompProfile и др. В Kubernetes они задаются в манифесте и передаются в runtime; аналог флагов Docker (`--user`, `--cap-drop`). Позволяет запускать контейнер от non-root, ограничивать capabilities и т.д. без изменения образа.

### Container runtime (CRI)

**Container runtime** в Kubernetes — компонент на ноде, который по запросу kubelet пуллит образ и запускает контейнер (создаёт и управляет контейнерами по CRI — Container Runtime Interface). Типичные реализации: **containerd** (с runc), **CRI-O**. Раньше использовался **Docker** через dockershim (kubelet вызывал Docker daemon); dockershim удалён, сейчас используется containerd или CRI-O. Образы те же (OCI); отличия — в настройке и в том, что демона Docker на ноде может не быть.

---

## Docker-образ в Kubernetes

Образ, собранный через Docker (или BuildKit) и запушенный в любой OCI-совместимый registry, указывается в Pod/Deployment так:

```yaml
spec:
  containers:
    - name: app
      image: myregistry.com/myapp:v1.0.0
      # image: myregistry.com/myapp@sha256:abc123...  # по digest
```

Kubelet передаёт запрос на pull и run в container runtime (containerd/CRI-O); runtime пуллит образ и создаёт контейнер. Никакой «конвертации» образа не требуется — формат OCI общий.

!!! tip "Практика"

    В production указывайте образ по тегу версии (v1.0.0) или по digest, чтобы деплой был воспроизводимым. imagePullPolicy при необходимости задавайте явно (например, IfNotPresent для ускорения при гарантированно обновлённом образе на ноде).
    
---

## ENTRYPOINT/CMD и command/args

В образе:

- **ENTRYPOINT** — исполняемый файл/команда.
- **CMD** — аргументы по умолчанию для ENTRYPOINT.

В поде:

- **command** — переопределяет ENTRYPOINT образа целиком.
- **args** — переопределяет CMD (аргументы для ENTRYPOINT) или аргументы для command, если задан command.

| В поде   | ENTRYPOINT образа | CMD образа |
|----------|-------------------|------------|
| Не задано | Используется      | Используется |
| Только args | Используется   | Заменён на args |
| command  | Заменён на command | Игнорируется (если не задан args, аргументов нет) |
| command + args | command как программа | args как аргументы |

Пример: образ с ENTRYPOINT ["/app/server"] и CMD ["--port", "8080"]. В поде можно передать другой порт без пересборки:

```yaml
containers:
  - name: app
    image: myapp:v1.0
    args:
      - "--port"
      - "9090"
```

ENTRYPOINT остаётся /app/server, аргументы будут --port 9090.

Полная замена команды (например, запуск shell для отладки):

```yaml
containers:
  - name: app
    image: myapp:v1.0
    command: ["/bin/sh", "-c"]
    args:
      - "sleep 3600"
```

---

## imagePullPolicy

- **Always** — перед запуском контейнера всегда обращаться к registry и при изменении тега подтягивать новый образ. По умолчанию для тега `:latest` или если тег не указан.
- **IfNotPresent** — пуллить только если образа с таким именем и тегом ещё нет на ноде. По умолчанию для любого конкретного тега (например, v1.0.0).
- **Never** — использовать только локальный образ; если его нет, под не стартует. Для воздушных окружений или предзагруженных образов.

```yaml
containers:
  - name: app
    image: myapp:v1.0.0
    imagePullPolicy: IfNotPresent
```

При использовании digest (`image: myapp@sha256:...`) образ однозначно определён; политика чаще IfNotPresent, чтобы не дергать registry без необходимости.

---

## securityContext

Настройки безопасности контейнера задаются на уровне пода (pod securityContext) и контейнера (container securityContext). Примеры:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
    - name: app
      image: myapp:v1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

- **runAsUser / runAsNonRoot** — от какого пользователя запускать процесс (безопаснее non-root).
- **readOnlyRootFilesystem** — корень ФС только для чтения; запись только в смонтированные тома.
- **capabilities.drop: ALL** — убрать все лишние capabilities.

В Docker аналоги задаются флагами `--user`, `--read-only`, `--cap-drop`. В Kubernetes это декларативно в манифесте и единообразно для оркестратора.

---

## Отличия container runtime: containerd, CRI-O vs Docker daemon

На ноде Kubernetes контейнеры запускает не демон Docker (dockerd), а runtime по CRI:

- **containerd** — тот же движок, который использует Docker под капотом; образы в формате OCI, совместимы с Docker. На ноде нет команды `docker` (если не установлен отдельно), но образы те же.
- **CRI-O** — лёгкий runtime, совместимый с OCI и CRI; образы из тех же registry пуллируются так же.

Общие моменты:

- Образы собираются так же (docker build, push в registry) и в Kubernetes используются без изменений.
- Нет docker run/docker ps на ноде — управление контейнерами через kubectl и API сервера.
- Логи, метрики, сеть — зона ответственности Kubernetes (логи через kubelet, сеть через CNI). Отладка на ноде: crictl (для containerd/CRI-O) или установленный Docker CLI для совместимости (если образы те же, контейнеры видны через crictl).

!!! example "Production"

    CI собирает образ через Docker/BuildKit, пушит в корпоративный registry. В манифестах Kubernetes указывается тот же образ (тег или digest). imagePullPolicy и securityContext задаются в шаблонах (Helm, Kustomize). На нодах — containerd или CRI-O; образы и контейнеры совместимы с тем, что вы тестировали локально в Docker.
    
---

## Паттерны использования

| Паттерн | Описание |
|--------|----------|
| **Один образ — Docker и Kubernetes** | Собирать один раз, пушить в registry; использовать в Compose локально и в K8s в кластере. |
| **Переопределение через args** | Менять порт, конфиг или флаги приложения через args в поде, не пересобирая образ. |
| **securityContext в манифесте** | Запуск от non-root, drop capabilities, read-only root — задавать в Kubernetes, образ остаётся переносимым. |
| **Тег или digest в production** | Не использовать latest; фиксировать версию или digest для воспроизводимости. |

---

## Антипаттерны

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Жёстко полагаться на docker на ноде** | В K8s нода может быть без Docker; контейнеры управляются через CRI. | Использовать kubectl и crictl для отладки на ноде. |
| **Игнорировать securityContext** | Запуск от root и лишние capabilities увеличивают риски. | Задавать runAsNonRoot, drop capabilities в поде. |
| **Путать command и args** | command заменяет ENTRYPOINT; args заменяют CMD. | Читать справку по полям и проверять итоговую команду в поде. |

---

## Дополнительные материалы

- [Kubernetes — Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Kubernetes — Container lifecycle hooks and command/args](https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)
- [imagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/#updating-images)
- [Configure a Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [CRI — Container Runtime Interface](https://github.com/kubernetes/cri-api)
- [containerd](https://containerd.io/)
- [CRI-O](https://cri-o.io/)
