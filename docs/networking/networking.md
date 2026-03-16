# Networking
![Networking[200]](./images/welcome.png)
Материалы по сетевым технологиям: модели OSI и TCP/IP, L2/L3, практическая диагностика. Разделы содержат определения, примеры, практику и ссылки на дополнительное чтение.

## Разделы

- [**1. Базовая теория сетей**](topic-1-basis.md) — модели OSI и TCP/IP, инкапсуляция, Ethernet и L2 (MAC, ARP), Unicast/Broadcast/Multicast, VLAN, MTU, STP; практика: ip link, arp, tcpdump.

- [**2. IP-сети и маршрутизация**](topic-2-ip-routing.md) — IPv4/IPv6, CIDR и подсети, частные/публичные адреса, NAT (SNAT, DNAT, PAT), CGNAT, таблица маршрутизации, default route, ECMP; практика: ip addr, ip route, traceroute; почему под в Kubernetes недоступен извне без Service/Ingress.

- [**3. TCP и UDP**](topic-3-tcp-udp.md) — TCP: 3-way handshake, FIN vs RST, окно, MSS, ретрансмиты, TIME_WAIT/CLOSE_WAIT, Nagle и Delayed ACK; UDP: когда используется, QUIC (HTTP/3); практика: ss, netstat, tcpdump tcp.

- [**4. DNS**](topic-4-dns.md) — рекурсивный и авторитативный сервер, A/AAAA/CNAME/TXT/SRV, TTL и кэш, split-horizon, DNS в Kubernetes; практика: dig +trace, dig @8.8.8.8, nslookup; почему работает по IP, но не по hostname.

- [**5. HTTP и HTTPS (L7)**](topic-5-http-https.md) — HTTP: методы, статусы (3xx, 4xx, 5xx), заголовки, Keep-Alive; HTTPS/TLS: handshake, SNI, цепочка сертификатов, mTLS, ALPN; практика: curl -v, openssl s_client.

- [**6. Linux Networking**](topic-6-linux-networking.md) — сетевой стек (NIC, ядро, сокеты), инструменты (ip, ss, tcpdump, conntrack, ethtool), firewall (iptables/nftables, цепочки, stateful/conntrack); как пакет проходит INPUT/FORWARD/OUTPUT; почему порт открыт, но соединения нет.

- [**7. Load Balancing и Proxy**](topic-7-load-balancing-proxy.md) — балансировка L4 vs L7, round-robin/least connections, health checks; reverse/forward/transparent proxy; примеры: Nginx, HAProxy, Envoy.

- [**8. Kubernetes Networking**](topic-8-kubernetes-networking.md) — Pod IP, Service (ClusterIP, NodePort, LoadBalancer), kube-proxy (iptables/ipvs), CNI (Calico, Cilium, Flannel), Ingress и L7, NetworkPolicy (allow/deny, default deny); как под из namespace A ходит в под из B; почему NetworkPolicy не работает.

- [**9. Observability и отладка сети**](topic-9-observability-troubleshooting.md) — инструменты (tcpdump, Wireshark, mtr, curl, iperf); подход по шагам: проверка DNS, маршрутизации, firewall, L7 для быстрого поиска root cause.

*(Дальнейшие разделы будут добавляться по мере подготовки материалов.)*
