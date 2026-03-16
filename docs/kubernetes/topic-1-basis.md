# 1. База: Linux, сеть и контейнеры

**Цель:** понимать, почему Kubernetes работает именно так. Этот раздел закрывает необходимую базу: Linux (namespaces, cgroups, firewall), сеть (TCP/IP, DNS, NAT, балансировка L4/L7, TLS/mTLS) и контейнеры (образы, ENTRYPOINT/CMD, multi-stage, безопасность образов, OCI runtime). Без этого сложно осознанно работать с подами, сервисами и Ingress.

---

## Linux и сетевая база (обязательно)

### Namespaces и cgroups

Kubernetes запускает контейнеры как процессы на узлах; изоляция и ограничение ресурсов обеспечиваются механизмами ядра Linux. **Namespaces** изолируют процессы, сеть, монтирования, UTS, IPC (и при необходимости пользователей). **Cgroups** ограничивают CPU, память и I/O группы процессов. kubelet и container runtime (containerd, CRI-O) создают для каждого контейнера нужные namespace’ы и записывают лимиты в cgroups. Понимание нужно, чтобы объяснить, «где живёт» под и почему лимиты resources в Pod — это именно cgroups на узле.

См. разделы [Linux: 9. Linux и контейнеры](../linux/topic-9-linux-containers.md) и [Docker: 1. Базис](../docker/topic-1-basis.md).

### iptables / nftables (базовое понимание)

Service в Kubernetes (ClusterIP, NodePort) и часть Ingress реализуются через правила **iptables** или **ipvs** на каждой ноде (kube-proxy). Пакет к виртуальному IP сервиса перенаправляется на один из эндпоинтов (под). Базовое понимание цепочек (INPUT, FORWARD, NAT) и того, что kube-proxy меняет правила при изменении Service/Endpoints, помогает при отладке «сервис не резолвится» или «трафик не доходит до пода».

См. [Networking: 6. Linux Networking](../networking/topic-6-linux-networking.md).

### TCP/IP, DNS, NAT, Load Balancing

- **TCP/IP** — как пакеты доставляются до пода и обратно; порты, маршрутизация.
- **DNS** — в кластере CoreDNS резолвит имена сервисов (`<svc>.<ns>.svc.cluster.local`) и внешние имена; при проблемах «не резолвится имя» смотреть resolv.conf в поде и работу CoreDNS.
- **NAT** — исходящий трафик из пода часто выходит через SNAT на IP узла; входящий к Service — DNAT с ClusterIP на IP пода.
- **Load Balancing** — L4 (по IP/порту) vs L7 (по HTTP host/path). Service по умолчанию — L4; Ingress даёт L7 (маршрутизация по URL, host, TLS).

См. разделы [Networking: 2. IP-сети и маршрутизация](../networking/topic-2-ip-routing.md), [4. DNS](../networking/topic-4-dns.md), [7. Load Balancing и Proxy](../networking/topic-7-load-balancing-proxy.md).

### L7 vs L4 балансировка

- **L4** — решение по IP и порту; один виртуальный IP/порт → пул бэкендов; не видно HTTP. Так устроен Kubernetes Service (ClusterIP, NodePort).
- **L7** — решение по заголовкам HTTP (Host, path, заголовки); один порт 80/443 — много приложений по правилам. Так устроен Ingress и Ingress Controller (Nginx, Traefik и т.д.). Понимание разницы нужно при выборе Service vs Ingress и при настройке правил маршрутизации.

### TLS и mTLS (handshake, цепочка сертификатов)

- **TLS** — шифрование и аутентификация сервера: handshake, проверка цепочки сертификатов, SNI для выбора виртуального хоста. Ingress часто терминирует TLS на контроллере; сертификаты хранятся в Secret или выдаются через cert-manager.
- **mTLS** — взаимная аутентификация: клиент тоже предъявляет сертификат. В сервисных сетках (Istio, Linkerd) mTLS между подами включают для шифрования и идентификации. Понимание handshake и цепочки нужно при ошибках «certificate verify failed» и настройке Ingress/сервисной сетки.

См. [Networking: 5. HTTP и HTTPS (L7)](../networking/topic-5-http-https.md).

---

## Контейнеры

Kubernetes управляет **контейнерами** (образами, запуском, лимитами). Без чёткого понимания образов и runtime сложно правильно описывать Pod и отлаживать падения контейнеров.

### Docker: слои образа (image layers)

Образ собирается из **слоёв** (layers); каждый слой — набор изменений файловой системы. Неизменяемые слои кэшируются и переиспользуются между образами и контейнерами. В Kubernetes образ образа указывается в `spec.containers[].image`; при pull kubelet и runtime скачивают слои по необходимости. Понимание слоёв помогает при оптимизации размера образа и времени деплоя.

### ENTRYPOINT vs CMD

- **ENTRYPOINT** — команда, которая всегда выполняется при запуске контейнера.
- **CMD** — аргументы по умолчанию к ENTRYPOINT (или единственная команда, если ENTRYPOINT не задан).

В Kubernetes поле **command** в манифесте переопределяет ENTRYPOINT образа, **args** — CMD (или аргументы к command). Важно не путать: если в образе ENTRYPOINT — скрипт, а в Pod задать только args, они передадутся скрипту; если задать command — ENTRYPOINT образа игнорируется.

См. [Docker: 3. Образы](../docker/topic-3-images.md), [4. Контейнеры как процессы](../docker/topic-4-containers.md).

### Multi-stage builds

**Multi-stage** — несколько стадий в одном Dockerfile; итоговый образ собирается из последней стадии. В первых стадиях компилируют код, в последней копируют только артефакты; в образ не попадают компиляторы и лишние зависимости. Образы для Kubernetes часто собирают multi-stage, чтобы уменьшить размер и поверхность атаки.

### Безопасность образов (non-root, distroless)

- **Non-root** — процесс в контейнере работает от непривилегированного пользователя (USER в Dockerfile, securityContext.runAsNonRoot в Pod). Снижает последствия при компрометации контейнера.
- **Distroless** (или минимальная база типа Alpine) — в образе нет shell и лишних пакетов; меньше уязвимостей и меньше размер. Для отладки «зайти в контейнер» тогда возможна только через ephemeral container или sidecar.

Эти практики напрямую связаны с securityContext в Pod (runAsUser, readOnlyRootFilesystem и т.д.) и с политиками безопасности кластера.

См. [Docker: 8. Безопасность](../docker/topic-8-security.md).

### OCI runtime (containerd, runc)

Kubernetes не привязан к Docker как к runtime; он использует **CRI** (Container Runtime Interface). На узле обычно работает **containerd** (или CRI-O), который по CRI получает запросы от kubelet и запускает контейнеры через **runc** (OCI runtime). Образы остаются в формате OCI (тот же, что и у Docker); Docker при этом может быть лишь клиентом для сборки и push, а в кластере образы тянут и запускают containerd. Понимание нужно, чтобы не путать «Docker не установлен на ноде» с «образ не подтягивается» (образы качает containerd из registry по imagePullSecrets и настройкам).

См. [Docker: 2. Архитектура](../docker/topic-2-architecture.md), [13. Docker и Kubernetes](../docker/topic-13-kubernetes.md).

---

## Зачем это для Kubernetes

Без этой базы сложно:

- Объяснить, почему под не стартует (образ, ресурсы, securityContext, образ без shell).
- Понять, как трафик попадает в под (Service → kube-proxy → DNAT → под, или Ingress → L7 → Service → под).
- Настроить TLS в Ingress и разобрать ошибки сертификатов.
- Выбрать между Service (L4) и Ingress (L7) и корректно задать command/args в Pod.

Рекомендуется закрыть пробелы по Linux (namespaces, cgroups), сети (DNS, NAT, балансировка) и контейнерам (образы, ENTRYPOINT/CMD, безопасность) до углубления в манифесты и контроллеры Kubernetes.

---

## Дополнительные материалы

- [Linux: 9. Linux и контейнеры](../linux/topic-9-linux-containers.md)
- [Docker: 1. Базис](../docker/topic-1-basis.md)
- [Docker: 3. Образы](../docker/topic-3-images.md)
- [Networking: 6. Linux Networking](../networking/topic-6-linux-networking.md)
- [Networking: 5. HTTP и HTTPS (L7)](../networking/topic-5-http-https.md)
- [Kubernetes — Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
