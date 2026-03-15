# 7. Docker Compose

**Цель темы:** описывать локальные и CI-окружения с помощью Docker Compose: структура `docker-compose.yml` (v3+), сервисы, сети и тома, зависимости между сервисами, переменные окружения, профили, override-файлы и стратегии ожидания готовности (healthcheck, wait).

---

## Определения терминов

### Docker Compose

**Docker Compose** — инструмент для описания и запуска многоконтейнерных приложений. Конфигурация в YAML (`docker-compose.yml`); один проект — набор сервисов, сетей и томов. Команды `docker compose up`, `docker compose down` и др. создают и управляют контейнерами по этому описанию. Подходит для локальной разработки, тестов и CI.

### Сервис (service)

**Сервис** в Compose — один образ и конфигурация запуска (порты, тома, переменные, зависимости). Каждый сервис при `up` превращается в один или несколько контейнеров (при `replicas`/`deploy.replicas` — несколько). Сервисы обращаются друг к другу по **имени сервиса** как по DNS (внутри одной сети проекта).

### Проект (project)

**Проект** — пространство имён Compose: имя берётся из имени каталога или из `-p` / переменной `COMPOSE_PROJECT_NAME`. Все ресурсы (контейнеры, сети, тома) помечаются этим именем. Разные проекты изолированы друг от друга.

### Override-файл

**Override** — дополнительный файл (по умолчанию `docker-compose.override.yml`), который автоматически объединяется с основным при запуске. Используется для локальных переопределений (порты, bind mount исходников) без изменения основного файла, общего для CI и команды.

### Профили (profiles)

**Профили** — теги для сервисов; при `docker compose up` без указания профиля запускаются только сервисы без профиля. С `--profile <name>` добавляются сервисы с этим профилем. Удобно для опциональных сервисов (тесты, инструменты, dev-only).

---

## Структура docker-compose.yml (v3+)

Базовые ключи верхнего уровня:

- **services** — описание сервисов (образ, порты, тома, env, команда и т.д.).
- **networks** — пользовательские сети (опционально; по умолчанию создаётся одна сеть проекта).
- **volumes** — именованные тома (опционально; при использовании в сервисах создаются автоматически при необходимости).

Версия формата (version) в Compose V2 не обязательна; рекомендуется не указывать или указывать `"3"` / `"3.8"` для совместимости.

```yaml
services:
  app:
    image: myapp:latest
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
    volumes:
      - appdata:/data
    networks:
      - backend

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  appdata:
  pgdata:

networks:
  backend:
    driver: bridge
```

---

## Сервисы: образ, порты, тома, env

### Образ и сборка

```yaml
services:
  app:
    image: myregistry/myapp:1.0
    # или сборка из контекста:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
```

При наличии `build` и `image` образ сначала собирается, затем тегируется как `image` (удобно для push).

### Порты

```yaml
ports:
  - "8080:80"
  - "127.0.0.1:9090:90"
```

Формат «хост:контейнер»; привязка к localhost только для указанного адреса.

### Тома

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
  - ./config:/app/config:ro
```

Именованный том `pgdata` и bind mount каталога `./config` в режиме только чтения.

### Переменные окружения

```yaml
environment:
  - NODE_ENV=production
  - DB_HOST=db
env_file:
  - .env
  - .env.stage
```

`env_file` подставляет переменные из файла; переменные из `environment` переопределяют одноимённые из файла.

---

## depends_on и ограничения

### Базовый depends_on

```yaml
services:
  app:
    depends_on:
      - db
      - redis
  db:
    image: postgres:15
  redis:
    image: redis:7-alpine
```

Compose запускает `db` и `redis` перед `app`. По умолчанию **не** ждёт готовности приложения в зависимом сервисе — только создание контейнера.

### Условие готовности (condition)

В формате Compose v2/v3 с расширенным синтаксисом можно указать ожидание healthcheck:

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
```

- **service_started** — контейнер создан и запущен.
- **service_healthy** — healthcheck прошёл (нужен `healthcheck` у зависимого сервиса).

!!! tip "Практика"

    Для БД и кэша всегда задавайте **healthcheck** и используйте `condition: service_healthy` в `depends_on`, чтобы приложение не стартовало до готовности зависимостей.
    
---

## env_file и разные окружения (dev/stage)

Отдельные файлы для окружений:

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    env_file:
      - .env
```

- `.env` — общие переменные (не коммитить секреты в репо; использовать `.env.example` как шаблон).
- Локально: `.env`, в CI: `.env.ci`, для stage: `env_file: [.env.stage]` или переопределение через override.

Override для разработки:

```yaml
# docker-compose.override.yml (не коммитить секреты)
services:
  app:
    build: .
    volumes:
      - .:/app
    env_file:
      - .env.local
```

Разные наборы: `docker compose --env-file .env.stage up` подставит переменные из `.env.stage` в шаблон `${VAR}` в compose-файле; для значений внутри контейнера по-прежнему используется `env_file` сервиса.

---

## Профили (profiles)

Сервисы с профилем запускаются только при указании этого профиля:

```yaml
services:
  app:
    image: myapp
  db:
    image: postgres:15
  tests:
    image: myapp
    profiles:
      - test
    command: ["npm", "test"]
  dev-tools:
    image: devtools
    profiles:
      - dev
```

```bash
docker compose up -d
# поднимает app и db

docker compose --profile test up tests
# поднимает app, db и tests

docker compose --profile dev up
# поднимает app, db и dev-tools
```

Удобно для опциональных сервисов (миграции, тесты, вспомогательные утилиты).

---

## Override-файлы

По умолчанию Compose объединяет `docker-compose.yml` и `docker-compose.override.yml`. В override переопределяют порты, тома, команду и т.д. для локальной разработки.

```yaml
# docker-compose.override.yml
services:
  app:
    build: .
    volumes:
      - .:/app
    ports:
      - "3000:8080"
```

Запуск: `docker compose up` — подхватываются оба файла. В CI часто используют только основной файл: `docker compose -f docker-compose.yml up` (без override) или отдельный `docker-compose.ci.yml`.

Явное указание нескольких файлов:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Порядок важен: последующие файлы переопределяют предыдущие.

---

## Healthcheck и стратегии ожидания

### Healthcheck в Compose

```yaml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
```

Пока healthcheck не станет healthy, сервис считается неготовым. Это используется при `depends_on` с `condition: service_healthy` и при `docker compose up` (логи могут показывать ожидание).

### Ожидание в CI/скриптах

Compose не ждёт «готовности приложения» по умолчанию — только старт контейнера или healthcheck. В CI после `docker compose up -d` часто добавляют цикл ожидания:

```bash
docker compose up -d
until docker compose exec -T db pg_isready -U postgres; do sleep 2; done
# или с retry для HTTP:
# curl -f http://localhost:8080/health || exit 1
```

Либо использовать `condition: service_healthy` и дать Compose время поднять сервисы с healthcheck.

---

## Практика: full-stack (backend + db + cache)

Пример минимального стека:

```yaml
services:
  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - appnet

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - appnet

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - appnet

volumes:
  pgdata:

networks:
  appnet:
    driver: bridge
```

Запуск: `docker compose up -d`. Backend обращается к `db` и `redis` по имени сервиса; зависимости ждут healthy.

---

## Паттерны использования

| Паттерн | Описание |
|--------|----------|
| **Healthcheck для БД и кэша** | Всегда задавать healthcheck и использовать `depends_on` с `condition: service_healthy`. |
| **Один общий .env.example** | Шаблон переменных без секретов; реальные секреты в .env (в .gitignore) или в CI variables. |
| **Override для локальной разработки** | Основной файл — общий; в override — build, bind mount исходников, лишние порты. |
| **Профили для опциональных сервисов** | Тесты, миграции, dev-инструменты — через профили, чтобы не поднимать их в production-like сценариях. |

---

## Антипаттерны

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Полагаться только на depends_on без condition** | Приложение может стартовать до готовности БД и падать. | Добавить healthcheck и `condition: service_healthy`. |
| **Коммитить .env с секретами** | Утечка учётных данных. | .env в .gitignore; использовать .env.example и CI secrets. |
| **Один compose для всего без профилей** | В CI и локально поднимается лишнее. | Вынести опциональное в профили или отдельные файлы. |
| **Жёстко заданные порты в одном файле** | Конфликты при нескольких проектах на одной машине. | Переменные окружения для портов или разные override. |

---

## Примеры из production

### CI: один файл без override

В пайплайне вызывают `docker compose -f docker-compose.yml up -d` (или отдельный `docker-compose.ci.yml`), без override. Образы собираются или тянутся из registry; переменные из CI (env или env_file из секретов). После up — ожидание healthcheck или явный wait-скрипт, затем тесты.

### Разные окружения (dev/stage)

Один базовый `docker-compose.yml`; для stage/prod — `docker-compose.prod.yml` с другими образами, лимитами и без bind mount исходников. Запуск: `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d`. Либо один файл и разные `--env-file` для подстановки имён образов и портов.

### Полный стек для локальной разработки

Backend + DB + cache + фронт (если нужен); в override — монтирование кода и hot-reload. Один `docker compose up` поднимает всё необходимое; зависимости с healthcheck исключают гонки при старте.

---

## Дополнительные материалы

- [Docker Compose — обзор](https://docs.docker.com/compose/)
- [Compose file reference](https://docs.docker.com/compose/compose-file/)
- [depends_on](https://docs.docker.com/compose/compose-file/05-services/#depends_on)
- [Profiles](https://docs.docker.com/compose/profiles/)
- [Override file](https://docs.docker.com/compose/extends/)
- [Environment variables](https://docs.docker.com/compose/environment-variables/)
- [Healthcheck in Compose](https://docs.docker.com/compose/compose-file/05-services/#healthcheck)
