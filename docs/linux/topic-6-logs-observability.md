# 6. Логи и observability

---
journald, rsyslog, logrotate, логи ядра и базовый auditd — для быстрого поиска причин проблем и диагностики «сервис не стартует».

---

## journald

**journald** — подсистема логирования systemd. Собирает сообщения от ядра, systemd, сервисов и приложений, пишущих в stdout/stderr, в единый журнал (обычно бинарный, в /var/log/journal/ или в памяти). Запросы — через **journalctl**.

### Основные команды

```bash
# Все логи (с загрузки)
journalctl

# Логи конкретного unit (сервиса)
journalctl -u nginx.service

# Последние строки и follow (как tail -f)
journalctl -f
journalctl -u nginx -f

# За период (сегодня, с даты, за последний час)
journalctl --since today
journalctl --since "2024-01-15" --until "2024-01-16"
journalctl --since "1 hour ago"

# Только ошибки и выше (priority)
journalctl -p err
journalctl -u nginx -p err

# Вывод в обратном порядке (новые сверху)
journalctl -r
```

При «сервис не стартует» первым делом смотреть: `journalctl -u <unit> -n 100 --no-pager` — там обычно причина (ошибка конфига, порт занят, permission denied).

---

## rsyslog

**rsyslog** — классический демон системного логирования: принимает сообщения по протоколу syslog (UDP/TCP) и по правилам раскладывает их по файлам (например, /var/log/syslog, /var/log/auth.log). Многие приложения пишут в syslog; rsyslog может дублировать приём из journald (модуль imjournal) и записывать в файлы. Конфигурация — /etc/rsyslog.conf и файлы в /etc/rsyslog.d/. После правок: `sudo systemctl reload rsyslog`.

---

## logrotate

**logrotate** — ротация логов: переименование/архивирование и сжатие по расписанию или по размеру, удаление старых. Предотвращает заполнение диска логами. Конфиги — /etc/logrotate.conf (общие настройки) и /etc/logrotate.d/* (по приложениям).

### Пример конфигурации

Файл /etc/logrotate.d/myapp:

```
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 myapp myapp
    postrotate
        systemctl reload myapp
    endscript
}
```

- **daily** — ротация раз в день (альтернативы: weekly, size 100M).
- **rotate 7** — хранить 7 архивов.
- **compress** / **delaycompress** — сжимать предыдущий файл (текущий оставлять без сжатия для записи).
- **missingok** — не считать ошибкой отсутствие файла.
- **notifempty** — не ротировать пустой файл.
- **create** — права и владелец нового файла после ротации.
- **postrotate** — команда после ротации (например, переоткрыть лог в приложении).

Проверка конфига: `sudo logrotate -d /etc/logrotate.d/myapp` (dry run, вывод без изменений). Принудительный запуск: `sudo logrotate -f /etc/logrotate.conf`.

---

## Логи ядра (kernel logs)

Сообщения ядра (dmesg) попадают в ring buffer и в journald. Команды:

```bash
# Текущий буфер ядра
dmesg

# Только последние сообщения
dmesg -T   # с человекочитаемыми метками времени (если поддерживается)
dmesg | tail -50

# Через journalctl (логи ядра с загрузки)
journalctl -k
journalctl -k --since "1 hour ago"
```

При падении сервера, OOM, сбоях диска или драйвера смотреть dmesg и journalctl -k; часто там единственная подсказка.

---

## auditd (база)

**auditd** — демон аудита: записывает события по правилам (вход в систему, смена прав, доступ к файлам по заданным путям и т.д.). Логи — обычно /var/log/audit/audit.log. Запросы: **ausearch**, **aureport**. Правила — **auditctl** или постоянные в /etc/audit/rules.d/. Для DevOps достаточно понимать: auditd даёт детальный след по безопасности; при расследовании инцидентов и при настройке SELinux смотреть audit.log и ausearch. Включённость: `systemctl status auditd`.

---

## Практика

### Найти, почему сервис не стартует

```bash
# Логи unit’а за последний запуск
journalctl -u myservice.service -n 100 --no-pager

# С момента последней загрузки
journalctl -u myservice -b

# С подробными полями (например, exit code)
journalctl -u myservice -o verbose -n 20
```

В выводе искать: failed, error, permission denied, address already in use, exit code. По коду и сообщению исправлять конфиг, права, порт или зависимости (After=, Requires=).

### Настроить logrotate

1. Создать файл в /etc/logrotate.d/ (имя по приложению).
2. Указать путь к логам, период или размер (daily/size), rotate N, compress при необходимости.
3. При необходимости postrotate/endscript (reload приложения, чтобы переоткрыло лог).
4. Проверить: `logrotate -d /etc/logrotate.d/yourfile`.
5. Запуск по расписанию обычно через cron (cron.daily) или timer systemd; при необходимости запустить вручную: `logrotate -f /etc/logrotate.conf`.

### Отфильтровать логи за конкретный период

**journalctl:**

```bash
journalctl --since "2024-01-15 09:00:00" --until "2024-01-15 18:00:00"
journalctl -u nginx --since yesterday
journalctl --since "2 hours ago"
```

**Файловые логи (syslog, приложение):** использовать время в строке лога и стандартные утилиты:

```bash
# Если в логах есть ISO или формат с датой
grep "2024-01-15" /var/log/syslog
# или с awk/sed по полю времени
```

Для единообразия по времени удобно централизованное хранение (ELK, Loki и т.д.) с индексом по времени.

!!! tip "Практика"

    При любом инциденте: 1) journalctl -u <service> — почему не стартует или что упало; 2) journalctl -k и dmesg — если подозрение на ядро/железо; 3) знать, где лежат логи приложения (файлы или только journald) и как их ротировать (logrotate).

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Сначала journalctl -u** | Для systemd-сервисов логи старта и ошибок в журнале. |
| **logrotate для всех долгоживущих логов** | Иначе /var/log заполнится. |
| **Использовать --since/--until** | Быстро сузить выборку по времени. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Ротировать логи без postrotate** | Приложение продолжит писать в переименованный файл или в удалённый. | Вызывать reload/restart после ротации, чтобы переоткрыть лог. |
| **Игнорировать логи ядра при сбоях** | OOM, паника, ошибки диска видны в dmesg/journalctl -k. | После перезагрузки смотреть journalctl -k -b -1 (предыдущая загрузка). |

---

## Дополнительные материалы

- [journalctl(1)](https://man7.org/linux/man-pages/man1/journalctl.1.html)
- [rsyslog.conf](https://www.rsyslog.com/doc/)
- [logrotate(8)](https://man7.org/linux/man-pages/man8/logrotate.8.html)
- [dmesg(1)](https://man7.org/linux/man-pages/man1/dmesg.1.html)
- [auditd](https://linux.die.net/man/8/auditd)
