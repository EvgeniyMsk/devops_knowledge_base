# 12. Отладка и устранение неполадок

**Цель темы:** уметь диагностировать проблемы контейнеров, не ограничиваясь пересборкой: использовать docker logs, inspect, exec, stats, при необходимости nsenter и tcpdump, и методично разбирать типовые сценарии — контейнер сразу падает, приложение не принимает трафик, OOMKilled.

---

## Определения терминов

### Troubleshooting (устранение неполадок)

**Troubleshooting** — процесс поиска и устранения причины сбоя: сбор логов, проверка конфигурации и состояния контейнера, сети и ресурсов. В контексте Docker — использование CLI и утилит хоста для понимания, почему контейнер не стартует, падает или не отвечает на запросы.

### Debug-контейнер

**Debug-контейнер** — образ с отладочными утилитами (shell, curl, tcpdump, strace и т.д.), который запускают в той же сети или с теми же namespace, что и проблемный контейнер, чтобы проверить связность, DNS, порты или временно заменить команду контейнера для воспроизведения. В Kubernetes аналог — ephemeral debug container (`kubectl debug`).

---

## Основные команды диагностики

### docker logs

Просмотр stdout/stderr контейнера (то, что пишет основной процесс).

```bash
docker logs <container>
docker logs -f <container>        # поток в реальном времени
docker logs --tail 100 <container>
docker logs --since 10m <container>
docker logs -t <container>        # с метками времени
```

Если контейнер сразу падает — логи покажут последний вывод перед выходом (ошибка приложения, отсутствующий файл, неверный порт и т.д.).

### docker inspect

Полная информация о контейнере (конфиг, сеть, тома, состояние, PID на хосте, лимиты, env).

```bash
docker inspect <container>
docker inspect --format '{{.State.Status}}' <container>
docker inspect --format '{{.State.ExitCode}}' <container>
docker inspect --format '{{.State.Pid}}' <container>
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>
```

Полезно для проверки: какой образ, какие порты и тома, какой exit code при падении, есть ли лимиты.

### docker exec

Запуск команды внутри работающего контейнера. Для отладки — получить shell или выполнить проверки (curl, ping, просмотр файлов).

```bash
docker exec -it <container> sh
docker exec <container> cat /etc/config/app.json
docker exec <container> curl -s http://localhost:8080/health
```

Если контейнер не в состоянии «running», exec недоступен — тогда смотрят логи и inspect остановленного контейнера.

### docker stats

Использование CPU, памяти и сети контейнерами в реальном времени.

```bash
docker stats
docker stats --no-stream
```

Помогает увидеть утечку памяти, высокую нагрузку по CPU и какой контейнер потребляет ресурсы.

---

## nsenter

**nsenter** — утилита для входа в namespace другого процесса (на хосте). Позволяет «зайти» в network, PID, mount namespace контейнера без docker exec (например, если внутри контейнера нет shell).

Узнать PID процесса контейнера на хосте:

```bash
docker inspect --format '{{.State.Pid}}' <container>
```

Войти в network namespace (от имени root на хосте):

```bash
nsenter -t <pid> -n ip addr
nsenter -t <pid> -n curl http://localhost:8080
```

`-n` — network namespace; можно комбинировать с `-m` (mount), `-p` (PID) и т.д. На практике чаще используют `docker exec`, если в образе есть shell; nsenter нужен, когда exec невозможен (минимальный образ без shell).

---

## tcpdump внутри контейнера

Проверка сетевого трафика: доходят ли пакеты до контейнера, что уходит наружу.

Варианты:

1. **docker exec** в контейнер, где есть tcpdump (не все образы его содержат):
   ```bash
   docker exec <container> tcpdump -i eth0 -n port 80
   ```

2. **Временный контейнер с сетевым режимом проблемного** — подключить контейнер к той же сети и запустить tcpdump оттуда, либо использовать `--network container:<id>` (контейнер разделяет сеть с целевым):
   ```bash
   docker run --rm -it --network container:<target_id> nicolaka/netshoot tcpdump -i eth0 -n
   ```

На хосте можно снимать трафик с интерфейса veth, привязанного к контейнеру (нужен вывод `docker inspect` или `ip link`).

---

## Debug-образы и временные контейнеры

Образы с набором утилит для сети и отладки:

- **nicolaka/netshoot** — tcpdump, curl, dig, nslookup, iperf и др.
- **alpine** или **busybox** — минимальный shell, wget, nc.

Запуск в той же сети, что и проблемный сервис:

```bash
docker run --rm -it --network container:web nicolaka/netshoot
# или подключиться к сети проекта Compose:
docker run --rm -it --network myproject_default nicolaka/netshoot
```

Из такого контейнера проверяют: резолвится ли имя сервиса, доступен ли порт (curl, nc), как выглядит маршрутизация.

---

## Сценарий 1: контейнер стартует и сразу падает

### Что делать

1. **Логи:** `docker logs <container>` — последние строки перед выходом (ошибка приложения, отсутствующий файл, неверная конфигурация).
2. **Exit code:** `docker inspect --format '{{.State.ExitCode}}' <container>`. Ненулевой код — приложение или скрипт завершились с ошибкой.
3. **Запуск без detached:** `docker run --rm <image>` (без `-d`) — вывод в текущий терминал; при падении сразу видно сообщение.
4. **Переопределить команду:** `docker run --rm -it <image> sh` — войти в контейнер и вручную запустить команду из CMD/ENTRYPOINT, проверить пути, переменные, права.
5. **Проверить образ:** тот же Dockerfile и контекст — возможно, проблема в коде или в отсутствующем файле в образе.

Типичные причины: неверный путь к бинарнику или конфигу, отсутствие обязательной переменной окружения, приложение слушает только 127.0.0.1 вместо 0.0.0.0, недостаточно прав на запись в каталог.

---

## Сценарий 2: приложение не принимает трафик

### Что делать

1. **Контейнер в состоянии running:** `docker ps`.
2. **Порты:** `docker port <container>` и `docker inspect` — убедиться, что порт опубликован и привязан к ожидаемому интерфейсу (0.0.0.0 или 127.0.0.1).
3. **Слушает ли приложение:** из контейнера `docker exec <container> netstat -tlnp` или `ss -tlnp` (если есть) — порт в LISTEN и на 0.0.0.0, а не только на 127.0.0.1.
4. **Доступ изнутри:** `docker exec <container> curl -s http://localhost:PORT/health` — отвечает ли приложение локально.
5. **С хоста:** `curl http://localhost:PORT` (или с другого контейнера в той же сети по имени сервиса). Если с хоста не доходит — проверить iptables, firewall, привязку порта.
6. **Сеть и DNS:** из другого контейнера в той же сети — резолвится ли имя, доступен ли порт (debug-контейнер с curl/nc).
7. **Логи приложения и демона:** ошибки биндинга, TLS, конфигурации.

Типичные причины: приложение слушает 127.0.0.1; порт не опубликован или опубликован на другой; firewall на хосте; контейнер в другой сети и имя не резолвится.

---

## Сценарий 3: OOMKilled

Контейнер завершился с кодом 137 (SIGKILL после исчерпания лимита памяти).

### Что делать

1. **Подтвердить:** `docker inspect --format '{{.State.OOMKilled}}' <container>` — true; в логах/событиях может быть указана причина выхода.
2. **Лимит:** `docker inspect --format '{{.HostConfig.Memory}}' <container>` — какой лимит был задан.
3. **Оценка потребления:** при следующем запуске смотреть `docker stats` до падения; либо включить детальную метрику памяти (cAdvisor, метрики cgroups).
4. **Действия:** увеличить `--memory` для контейнера или оптимизировать приложение (утечки, избыточное кэширование). Добавить мониторинг и алерты на OOMKilled.

!!! tip "Практика"

    Всегда задавайте лимит памяти в production; при OOMKilled сначала оцените, достаточно ли лимита для нормальной работы приложения или есть утечка. Временное увеличение лимита — не заменяет исправление утечки.
    
---

## Паттерны использования

| Паттерн | Описание |
|--------|----------|
| **Сначала логи и inspect** | Логи дают причину падения; inspect — конфиг, exit code, OOMKilled. |
| **Воспроизведение без -d** | Запуск без detached для немедленного вывода при падении. |
| **exec и debug-контейнер** | Проверка изнутри (curl, порты, DNS) и из соседнего контейнера в той же сети. |
| **Проверка портов и binding** | Убедиться, что приложение слушает 0.0.0.0 и порт опубликован. |

---

## Антипаттерны

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Сразу пересобирать образ** | Не устраняет причину (конфиг, env, лимиты). | Сначала логи, inspect, воспроизведение. |
| **Игнорировать exit code** | Код 137 = OOM; 1 = ошибка приложения. | Смотреть State.ExitCode и State.OOMKilled. |
| **Не проверять binding** | Приложение на 127.0.0.1 недоступно снаружи. | Проверять netstat/ss и слушать 0.0.0.0. |

---

## Дополнительные материалы

- [docker logs](https://docs.docker.com/engine/reference/commandline/logs/)
- [docker inspect](https://docs.docker.com/engine/reference/commandline/inspect/)
- [docker exec](https://docs.docker.com/engine/reference/commandline/exec/)
- [docker stats](https://docs.docker.com/engine/reference/commandline/stats/)
- [nsenter](https://man7.org/linux/man-pages/man1/nsenter.1.html)
- [Netshoot — network troubleshooting container](https://github.com/nicolaka/netshoot)
- [Docker troubleshooting](https://docs.docker.com/config/containers/runmetrics/)
