# 14. Справочник: команды Docker CLI

**Цель:** быстрый обзор основных команд Docker CLI для повседневной работы и поиска нужной команды.

Подробное описание флагов и сценариев — в соответствующих разделах (контейнеры, образы, сеть, тома, Compose и т.д.). Здесь — краткая шпаргалка по группам команд.

---

## Контейнеры (containers)

| Команда | Назначение |
|---------|------------|
| `docker run` | Создать и запустить контейнер (`-d`, `-p`, `-v`, `-e`, `--name`, `--rm`, `--restart`, `--memory`, `--cpus`) |
| `docker start` | Запустить остановленный контейнер |
| `docker stop` | Остановить (SIGTERM, `-t` — таймаут) |
| `docker restart` | Перезапустить контейнер |
| `docker kill` | Принудительно завершить (SIGKILL по умолчанию, `--signal=`) |
| `docker pause` | Приостановить все процессы в контейнере |
| `docker unpause` | Снять паузу |
| `docker rm` | Удалить контейнер (`-f` принудительно, `-v` удалить анонимные тома) |
| `docker create` | Создать контейнер без запуска |
| `docker exec` | Выполнить команду в работающем контейнере (`-it` для интерактива) |
| `docker attach` | Присоединиться к stdin/stdout основного процесса |
| `docker wait` | Дождаться завершения контейнера, вернуть exit code |
| `docker rename` | Переименовать контейнер |
| `docker update` | Обновить лимиты ресурсов контейнера без пересоздания |
| `docker port` | Показать проброс портов контейнера |
| `docker top` | Показать процессы внутри контейнера |
| `docker diff` | Показать изменения в ФС контейнера относительно образа |
| `docker commit` | Создать образ из изменений в контейнере (не для production) |
| `docker export` | Экспорт ФС контейнера в tar (без метаданных образа); импорт — `docker import` |
| `docker cp` | Копировать файлы контейнер ↔ хост |

---

## Логи и инспекция контейнеров

| Команда | Назначение |
|---------|------------|
| `docker logs` | Вывод stdout/stderr контейнера (`-f`, `--tail`, `--since`, `-t`) |
| `docker inspect` | JSON конфигурация и состояние контейнера (`--format` для полей) |
| `docker stats` | Использование CPU/памяти/сети в реальном времени (`--no-stream`) |
| `docker events` | Поток событий демона (create, start, stop, kill и т.д.) |

---

## Образы (images)

| Команда | Назначение |
|---------|------------|
| `docker images` | Список локальных образов (`-a` — все, включая промежуточные) |
| `docker pull` | Скачать образ из registry |
| `docker push` | Отправить образ в registry |
| `docker build` | Собрать образ из Dockerfile (`-t` тег, `-f` путь к Dockerfile, `--build-arg`) |
| `docker buildx` | Расширенная сборка (multi-platform, cache backends) |
| `docker rmi` | Удалить образ (`-f` принудительно) |
| `docker tag` | Создать тег для образа (alias) |
| `docker save` | Сохранить образ в tar (полный образ со слоями; перенос между хостами) |
| `docker load` | Загрузить образ из tar (созданного через `docker save`) |
| `docker import` | Создать образ из tar с ФС (из `docker export` или rootfs) |
| `docker history` | Показать историю слоёв образа |
| `docker image prune` | Удалить неиспользуемые образы (`-a` — все) |

---

## Сети (networks)

| Команда | Назначение |
|---------|------------|
| `docker network ls` | Список сетей |
| `docker network create` | Создать сеть (`-d` driver: bridge, overlay, macvlan; `--subnet`, `--gateway`) |
| `docker network rm` | Удалить сеть |
| `docker network inspect` | Детали сети (контейнеры, IP) |
| `docker network connect` | Подключить контейнер к сети |
| `docker network disconnect` | Отключить контейнер от сети |
| `docker network prune` | Удалить неиспользуемые сети |

---

## Тома (volumes)

| Команда | Назначение |
|---------|------------|
| `docker volume ls` | Список томов |
| `docker volume create` | Создать том (`--driver`, `--opt`) |
| `docker volume rm` | Удалить том |
| `docker volume inspect` | Детали тома (Mountpoint и т.д.) |
| `docker volume prune` | Удалить неиспользуемые тома |

---

## Контекст сборки (buildx)

| Команда | Назначение |
|---------|------------|
| `docker buildx create` | Создать builder |
| `docker buildx use` | Переключить текущий builder |
| `docker buildx build` | Сборка с опциями (`--platform`, `--cache-from`, `--cache-to`, `--push`) |
| `docker buildx inspect` | Информация о builder |
| `docker buildx prune` | Очистить кэш buildx |

---

## Docker Compose

| Команда | Назначение |
|---------|------------|
| `docker compose up` | Создать и запустить сервисы (`-d` фоново, `--build` пересобрать) |
| `docker compose down` | Остановить и удалить контейнеры, сети, тома |
| `docker compose ps` | Список контейнеров проекта |
| `docker compose logs` | Логи сервисов (`-f`, имя сервиса) |
| `docker compose exec` | Выполнить команду в сервисе |
| `docker compose run` | Запустить одноразовый контейнер сервиса |
| `docker compose build` | Собрать образы сервисов |
| `docker compose pull` | Скачать образы |
| `docker compose push` | Отправить образы в registry |
| `docker compose config` | Проверить и вывести итоговый конфиг |
| `docker compose images` | Образы, используемые сервисами |
| `docker compose top` | Процессы в контейнерах сервисов |
| `docker compose restart` | Перезапустить сервисы |
| `docker compose stop` | Остановить сервисы |
| `docker compose start` | Запустить остановленные сервисы |
| `docker compose kill` | Принудительно завершить |
| `docker compose pause` / `unpause` | Приостановить / снять паузу |
| `docker compose rm` | Удалить остановленные контейнеры |

---

## Система и информация

| Команда | Назначение |
|---------|------------|
| `docker info` | Информация о демоне, хранилище, runtime |
| `docker version` | Версии CLI и daemon |
| `docker system df` | Использование диска (образы, контейнеры, тома) |
| `docker system prune` | Удалить неиспользуемые данные (`-a` образы, `--volumes` тома) |
| `docker system events` | События системы (аналог `docker events`) |

---

## Контекст и конфигурация

| Команда | Назначение |
|---------|------------|
| `docker context ls` | Список контекстов (куда подключается CLI) |
| `docker context use` | Переключить контекст |
| `docker context inspect` | Детали контекста |
| `docker login` | Войти в registry (сохраняет в `~/.docker/config.json`) |
| `docker logout` | Выйти из registry |
| `docker config` | Управление конфигурациями (Swarm) |

---

## Секреты (Swarm)

| Команда | Назначение |
|---------|------------|
| `docker secret create` | Создать секрет |
| `docker secret ls` | Список секретов |
| `docker secret inspect` | Детали секрета |
| `docker secret rm` | Удалить секрет |

---

## Swarm (оркестрация)

| Команда | Назначение |
|---------|------------|
| `docker swarm init` | Инициализировать swarm на узле |
| `docker swarm join` | Присоединить узел к swarm |
| `docker swarm leave` | Покинуть swarm |
| `docker swarm update` | Обновить параметры swarm |
| `docker node ls` | Список узлов |
| `docker node inspect` | Детали узла |
| `docker node update` | Обновить узел (availability, label) |
| `docker service create` | Создать сервис |
| `docker service ls` | Список сервисов |
| `docker service ps` | Задачи сервиса |
| `docker service scale` | Изменить число реплик |
| `docker service update` | Обновить сервис (rolling update) |
| `docker service logs` | Логи сервиса |
| `docker service rm` | Удалить сервис |
| `docker stack deploy` | Развернуть stack из compose-файла |
| `docker stack ls` | Список стеков |
| `docker stack ps` | Задачи стека |
| `docker stack rm` | Удалить стек |

---

## Плагины

| Команда | Назначение |
|---------|------------|
| `docker plugin ls` | Список плагинов |
| `docker plugin install` | Установить плагин |
| `docker plugin enable` / `disable` | Включить / отключить плагин |
| `docker plugin rm` | Удалить плагин |

---

## Trust (Docker Content Trust)

| Команда | Назначение |
|---------|------------|
| `docker trust sign` | Подписать образ |
| `docker trust revoke` | Отозвать подпись |
| `docker trust inspect` | Показать подписи образа |

---

## Прочее

| Команда | Назначение |
|---------|------------|
| `docker scan` | Сканирование образов на уязвимости (Docker Scout) |
| `docker manifest` | Работа с manifest list (multi-arch образы) |
| `docker checkpoint` | Экспериментально: checkpoint/restore контейнера |
| `docker init` | Экспериментально: init-контейнеры |

---

## Дополнительные материалы

- [Docker CLI reference](https://docs.docker.com/engine/reference/commandline/cli/)
- [docker run reference](https://docs.docker.com/engine/reference/commandline/run/)
- [Docker Compose CLI](https://docs.docker.com/compose/reference/)
- [Docker cheat sheet](https://docs.docker.com/get-started/docker_cheatsheet/)
