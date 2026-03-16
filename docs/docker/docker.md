# Docker
![Docker[200]](./images/welcome.png)
## Разделы

- [**1. Необходимый базис**](topic-1-basis.md) — почему Docker работает именно так: namespaces (pid, net, mount, user), cgroups (cpu, memory, blkio), overlayfs, сеть (bridge, veth, iptables, NAT, port mapping), PID 1, сигналы, зомби-процессы; контейнер vs VM; паттерны и антипаттерны; production.

- [**2. Архитектура Docker**](topic-2-architecture.md) — Docker CLI, daemon, containerd, runc, Docker API, OCI (image spec, runtime spec), Docker vs Podman; что происходит при `docker run nginx`, где «живёт» контейнер; паттерны и антипаттерны; production.

- [**3. Образы Docker (глубоко)**](topic-3-images.md) — Dockerfile (FROM, RUN, CMD, ENTRYPOINT, COPY vs ADD, ARG vs ENV, USER, WORKDIR, HEALTHCHECK, STOPSIGNAL), слои и кэш, best practices (multi-stage, alpine vs distroless, pinned versions, .dockerignore), формат OCI и multi-arch; практика сравнения сборок; паттерны и антипаттерны; production.

- [**4. Контейнеры как процессы**](topic-4-containers.md) — жизненный цикл: run, start, stop, kill; CMD vs ENTRYPOINT, exec vs attach; STDIN/STDOUT, exit codes, restart policies; PID 1 (сигналы, tini/dumb-init, зомби); паттерны и антипаттерны; вопросы на собеседовании; production.

- [**5. Сеть Docker**](topic-5-networking.md) — драйверы (bridge, host, none, overlay, macvlan), port mapping, контейнер↔контейнер и контейнер↔хост, DNS, --link (deprecated), iptables (DOCKER, NAT, FORWARD); вопросы на собеседовании; паттерны и антипаттерны; production.

- [**6. Тома и хранилище**](topic-6-volumes.md) — volumes vs bind mounts, tmpfs, volume drivers, права (UID/GID), резервное копирование и восстановление; практика: БД в контейнере, обновление без потери данных; паттерны и антипаттерны; production.

- [**7. Docker Compose**](topic-7-compose.md) — docker-compose.yml (v3+), services, networks, volumes, depends_on и ограничения, env_file, профили, override-файлы, healthcheck и ожидание готовности; практика: full-stack (backend + db + cache), разные env (dev/stage); паттерны и антипаттерны; production.

- [**8. Безопасность Docker**](topic-8-security.md) — контейнер ≠ VM, privileged, capabilities, seccomp, AppArmor/SELinux, rootless Docker, секреты, сканирование образов; вопросы на собеседовании (выход из контейнера, Docker socket); паттерны и антипаттерны; production.

- [**9. Registry и жизненный цикл образов**](topic-9-registry.md) — Docker Hub vs приватный registry, аутентификация, стратегия тегирования (immutable tags), garbage collection, продвижение образов (dev → prod); практика: поднять private registry и настроить auth; паттерны и антипаттерны; production.

- [**10. Docker в CI/CD**](topic-10-cicd.md) — Docker-in-Docker vs Docker-outside-of-Docker, BuildKit, кэш в CI (registry cache), multi-arch сборки, сканирование в пайплайне; типовые кейсы (медленный билд, падение только в CI); паттерны и антипаттерны; production.

- [**11. Docker в production**](topic-11-production.md) — почему не всегда достаточно «голого» Docker, Docker vs Kubernetes, лимиты ресурсов (CPU/память), logging drivers, мониторинг контейнеров, graceful shutdown; паттерны и антипаттерны; production.

- [**12. Отладка и устранение неполадок**](topic-12-troubleshooting.md) — docker logs, inspect, exec, stats, nsenter, tcpdump, debug-контейнеры; сценарии: контейнер сразу падает, приложение не принимает трафик, OOMKilled; паттерны и антипаттерны.

- [**13. Docker и Kubernetes (связка)**](topic-13-kubernetes.md) — образ Docker в Kubernetes, ENTRYPOINT/CMD и command/args, imagePullPolicy, securityContext, отличия container runtime (containerd, CRI-O); паттерны и антипаттерны; production.

- [**14. Справочник: команды Docker CLI**](topic-14-cli-reference.md) — быстрый обзор основных команд CLI: контейнеры, образы, сети, тома, buildx, Compose, система, контекст, секреты, Swarm, плагины, trust и прочее.
