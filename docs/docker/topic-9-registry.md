# 9. Registry и жизненный цикл образов

**Цель темы:** управлять образами в компании: понимать разницу между Docker Hub и приватным registry, настраивать аутентификацию, выбирать стратегию тегирования (в т.ч. immutable tags), организовывать сбор мусора и продвижение образов из dev в prod.

---

## Определения терминов

### Registry (реестр образов)

**Registry** — сервис, хранящий и раздающий образы по протоколу, совместимому с OCI/Docker (HTTP API v2). Клиент (Docker daemon, containerd, оркестратор) по имени образа (host/repository:tag) пуллит слои и манифест. Примеры: Docker Hub, GitLab Container Registry, Harbor, приватный **Distribution** (референсная реализация от Docker).

### Docker Hub

**Docker Hub** — публичный и коммерческий registry от Docker. Бесплатные лимиты на pull для анонимов и авторизованных пользователей; приватные репозитории и команды — по подписке. Подходит для публичных образов и для небольших команд; в компаниях часто используют свой registry для контроля и изоляции.

### Private registry (приватный реестр)

**Приватный registry** — развёрнутый внутри периметра (или в облаке с ограничением доступа) реестр образов. Образы не уходят во внешний интернет; можно настроить auth, сканирование, репликацию и политики тегирования. Реализации: **Distribution** (open source), Harbor, GitLab Container Registry, облачные (ECR, ACR, GCR).

### Тег (tag)

**Тег** — человекочитаемое имя версии образа (например, `latest`, `v1.0.0`, `main-abc1234`). Один и тот же образ может иметь несколько тегов. **Mutable tag** — тег может быть переназначен на другой образ (например, `latest` обновляется при каждой сборке). **Immutable tag** — после push тег не перезаписывают; новая версия = новый тег (например, по digest или по уникальной версии).

### Image promotion (продвижение образа)

**Продвижение** — перенос образа из окружения в окружение (dev → stage → prod) без пересборки. Один и тот же digest образа помечается тегами окружения или копируется между registry/проектами. Гарантирует, что в prod развёрнут тот же артефакт, что прошёл тесты в dev/stage.

### Garbage collection (сбор мусора в registry)

**Garbage collection** — удаление неиспользуемых слоёв и манифестов из хранилища registry. Слои, на которые больше не ссылается ни один манифест (ни один тег), можно удалить для экономии места. Запускается вручную или по расписанию (зависит от реализации registry).

---

## Docker Hub vs приватный registry

| Критерий | Docker Hub | Приватный registry |
|----------|------------|---------------------|
| Размещение | Внешний сервис | Свой хост / облако |
| Контроль доступа | Учётные записи Hub, организации | Своя auth (basic, token, LDAP и т.д.) |
| Лимиты | Ограничения на pull/хранилище | Зависит от инфраструктуры |
| Сканирование, политики | Платные возможности | Harbor, встроенное в GitLab и т.д. |
| Сетевой доступ | Нужен выход в интернет | Можно только внутри сети |

Для корпоративных образов и CI обычно разворачивают приватный registry (Distribution, Harbor, GitLab) и пушат туда образы после сборки; pull в production идёт из этого registry.

---

## Аутентификация

### Docker login

Учётные данные сохраняются в `~/.docker/config.json` (или в credential helper). Демон использует их при push/pull.

```bash
docker login registry.company.com
# Username: ci-user
# Password: <token>
```

Для CI используют токены или сервисные учётки с минимальными правами; пароль не в коде, а в переменных/секретах (DOCKER_USER, DOCKER_PASSWORD или DOCKER_AUTH).

### Basic auth в registry

Distribution поддерживает basic auth через reverse proxy (nginx, Apache) или через встроенную конфигурацию (htpasswd). Заголовок `Authorization: Basic <base64(user:password)>` проверяется перед доступом к репозиториям.

### Токены и интеграция с IdP

Harbor, GitLab Registry и облачные registry (ECR, ACR, GCR) интегрируются с OAuth2/OIDC и выдают короткоживущие токены. В CI получают токен по IAM-роли или по client credentials и делают `docker login` с этим токеном перед push/pull.

---

## Стратегия тегирования

### Mutable теги

- **latest** — последняя собранная версия (часто для dev). Не использовать для production: «latest» может измениться в любой момент.
- **main**, **develop** — привязка к ветке; образ пересобирается при каждом коммите, тег тот же. Удобно для dev; для prod не воспроизводимо.

### Immutable теги

- **Версия по семверу:** `v1.0.0`, `v1.0.1` — один тег = один образ навсегда. Подходит для релизов.
- **По коммиту/хешу:** `main-abc1234`, `sha-abc1234` — уникально и воспроизводимо для каждой сборки.
- **По digest:** образ идентифицируется по digest (sha256:...); тег не нужен для однозначности, но для удобства часто добавляют тег версии.

В production рекомендуется использовать только immutable теги (семвер или хеш коммита), чтобы один тег всегда указывал на один и тот же образ.

### Пример схемы

- **dev:** образы с тегом `main-<short_sha>` или `dev-latest` (mutable).
- **stage:** тот же образ после тестов помечается тегом `stage-<short_sha>` или копируется в проект «stage».
- **prod:** образ помечается тегом `v1.0.0` или `prod-<short_sha>`, деплой только по этому тегу.

---

## Garbage collection в registry (Distribution)

Distribution не удаляет данные автоматически. После удаления манифеста или тега (если registry поддерживает delete) слои остаются на диске до запуска GC.

1. Включить удаление в конфиге registry (storage.delete enabled).
2. Запустить сборку мусора в read-only режиме registry (переменная окружения или флаг).
3. Остановить registry, выполнить `registry garbage-collect`, снова запустить.

Документация: [Registry configuration](https://distribution.github.io/docs/configuration/) и [Garbage collection](https://distribution.github.io/docs/garbage-collection/). В Harbor и облачных registry GC настраивается через UI или API.

---

## Продвижение образов (dev → prod)

Идея: один образ собирается один раз (в CI при коммите или при создании тега), проходит тесты; в prod деплоится тот же образ (тот же digest), без пересборки.

Варианты:

1. **Один registry, разные теги:** образ `myapp:main-abc1234` после прохождения тестов тегируется как `myapp:v1.0.0` или `myapp:prod`. Деплой в prod использует этот тег.
2. **Разные проекты/репозитории в одном registry:** образ копируется из `registry/dev/myapp` в `registry/prod/myapp` с тегом версии (docker pull, docker tag, docker push или API copy).
3. **Разные registry:** образ пуллится из dev-registry и пушится в prod-registry после approval (через пайплайн или вручную).

Главное — не пересобирать образ для prod; использовать артефакт, уже проверенный в dev/stage.

---

## Практика: поднять private registry и настроить auth

### Запуск registry (без auth, для теста)

```bash
docker run -d -p 5000:5000 --restart=always --name registry \
  -v registry_data:/var/lib/registry \
  registry:2
```

Push/pull: тег вида `localhost:5000/myimage:tag`, на клиенте для HTTP (не HTTPS) в daemon может понадобиться `insecure-registries` для `localhost:5000`.

### Registry с basic auth (htpasswd)

Создать каталог с паролем:

```bash
mkdir auth
docker run --rm --entrypoint htpasswd httpd:2 -Bbn user password > auth/htpasswd
```

Запуск registry с монтированием конфига и auth:

```yaml
# config.yml (упрощённо)
version: 0.1
storage:
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
auth:
  htpasswd:
    realm: Registry
    path: /auth/htpasswd
```

```bash
docker run -d -p 5000:5000 --name registry \
  -v $(pwd)/auth:/auth \
  -v registry_data:/var/lib/registry \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM=Registry \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

Клиент: `docker login localhost:5000` (user/password), затем `docker push localhost:5000/myimage:tag`.

!!! tip "Практика"

    В production registry обслуживают по HTTPS (TLS); auth через токены или интеграцию с LDAP/OAuth предпочтительнее простого htpasswd. Для хранения данных registry используют постоянный volume или объектное хранилище (S3-совместимое) через драйвер storage.

---

## Паттерны использования

| Паттерн | Описание |
|--------|----------|
| **Immutable теги в prod** | Семвер или хеш коммита; один тег = один образ. |
| **Продвижение без пересборки** | Один образ — dev → stage → prod через теги или копирование. |
| **Приватный registry для корпоративных образов** | Контроль доступа, сканирование, единая точка pull. |
| **Регулярный GC** | Настраивать удаление и GC, чтобы не раздувать хранилище. |

---

## Антипаттерны

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Деплой по тегу latest в prod** | latest меняется; деплой невоспроизводим. | В prod только теги с версией или хешем. |
| **Пересборка образа для prod** | Отличие от протестированного артефакта, риск расхождений. | Продвигать один образ через окружения. |
| **Отсутствие auth в registry** | Любой с доступом к сети может пушить/пуллить. | Включать auth (basic, token, IdP). |
| **Игнорировать GC** | Рост диска из-за старых слоёв и удалённых тегов. | Включить delete и запускать GC по расписанию. |

---

## Примеры из production

### CI: сборка и push в приватный registry

Пайплайн собирает образ, тегирует по коммиту и по тегу (если релиз), логинится в корпоративный registry и пушит. Деплой в stage/prod пуллит образ из того же registry по выбранному тегу.

### Promotion через теги в одном registry

После успешных тестов в stage образ `myapp:main-abc1234` тегируется как `myapp:release-1.2.3`. Деплой в prod использует `myapp:release-1.2.3`. Пересборки нет; в prod ровно тот образ, что был в stage.

### Harbor: сканирование и политики

В Harbor включают сканирование уязвимостей при push; неподписанные образы или образы с критическими CVE не разрешают пуллить в production-проект. Replication настраивают для копирования образов между инстансами.

---

## Дополнительные материалы

- [Distribution (Registry) — документация](https://distribution.github.io/docs/)
- [Docker Hub](https://docs.docker.com/docker-hub/)
- [Deploy a registry server](https://docs.docker.com/registry/deploying/)
- [Registry configuration](https://distribution.github.io/docs/configuration/)
- [Garbage collection](https://distribution.github.io/docs/garbage-collection/)
- [Harbor](https://goharbor.io/docs/)
- [Image tagging best practices](https://docs.docker.com/engine/reference/commandline/tag/#description)
- [OCI Distribution Spec](https://github.com/opencontainers/distribution-spec)
