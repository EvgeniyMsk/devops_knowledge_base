# Linux

Linux — семейство Unix-подобных операционных систем на базе ядра Linux, включающих тот или иной набор утилит и программ проекта GNU, и, возможно, другие компоненты. Как и ядро Linux, системы на его основе создаются и распространяются в соответствии с моделью разработки свободного и открытого программного обеспечения.

![Linux[200]](./images/welcome.jpg)
## Разделы

- [**1. Архитектура Linux и основы ОС**](topic-1-architecture.md) — kernel space vs user space, syscalls (fork, exec, read, write), процессы и потоки, systemd и unit-файлы, targets (runlevels), FHS; практика: загрузка до multi-user.target, разбор unit-файла, strace.

- [**2. Файловая система и диски**](topic-2-filesystem-disks.md) — inodes, жёсткие и символьные ссылки, ext4 и XFS, mount/fstab, LVM (PV, VG, LV), RAID (0/1/5/10), квоты; практика: LVM и расширение без downtime, восстановление файла по inode, причина «No space left on device».

- [**3. Пользователи, права и безопасность**](topic-3-users-security.md) — UID/GID, chmod и setuid/setgid/sticky, ACL, sudo (NOPASSWD, secure_path), PAM, SELinux и AppArmor (концепции и базовое использование); практика: доступ через ACL, разбор «permission denied» без chmod 777.

- [**4. Процессы, CPU, память**](topic-4-processes-cpu-memory.md) — PID/PPID, переключение контекста, load average, OOM Killer, память (RSS, VSZ, page cache, swap), nice/renice, cgroups (база); практика: процесс, съедающий память, разница high load vs high CPU, ограничение CPU/памяти через cgroups.

- [**5. Сеть в Linux**](topic-5-networking.md) — стек TCP/IP, iproute2 (ip, ss), ARP и маршрутизация, NAT/conntrack, DNS (resolv.conf, nsswitch.conf), iptables/nftables (концепции); практика: «доступен локально, но не извне», проблема с DNS, ss vs netstat.

- [**6. Логи и observability**](topic-6-logs-observability.md) — journald, rsyslog, logrotate, логи ядра, auditd (база); практика: почему сервис не стартует, настройка logrotate, фильтр логов за период.

- [**7. Bash и shell scripting**](topic-7-bash-scripting.md) — пайпы и перенаправления, коды выхода, xargs, trap, подоболочки, переменные окружения, Bash strict mode; практика: скрипт проверки состояния сервиса, корректная обработка ошибок.

- [**8. Управление пакетами и сборка ПО**](topic-8-package-management.md) — apt, yum/dnf, приоритеты репозиториев, основы rpm и deb, динамическая и статическая линковка (ldd).

- [**9. Linux и контейнеры**](topic-9-linux-containers.md) — namespaces, cgroups, overlayfs, capabilities, chroot vs контейнер; практика: namespaces контейнера, ограничение ресурсов и сравнение с bare metal.

- [**10. Troubleshooting: подход production-инженера**](topic-10-troubleshooting-mindset.md) — типовые кейсы (тормозит, закончился диск, сервис не стартует, нет сети, упал после рестарта); алгоритм: CPU/память/диск, логи, сеть, права, последние изменения.

- [**11. Дополнительные материалы**](topic-11-advanced-topics.md) — практические разборы для интервью и эксплуатации: кто пишет в файл (`lsof`, `fuser`, `strace`).
