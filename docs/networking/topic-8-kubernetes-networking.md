# 8. Kubernetes Networking

---
Pod IP, Service (ClusterIP/NodePort/LoadBalancer), kube-proxy (iptables/ipvs), CNI, Ingress/L7 и NetworkPolicy + типовые ответы «как ходит трафик между namespace» и почему политики могут не работать.

---

## Основы

### Pod IP

Каждый под получает **собственный IP** из подсети, которую выделяет **CNI** (Container Network Interface). Этот IP уникален в пределах кластера; поды могут общаться друг с другом по этому IP без NAT. IP пода нестабилен: при пересоздании пода он меняется, поэтому для стабильного доступа используют **Service**.

### Service: ClusterIP, NodePort, LoadBalancer

**Service** — абстракция стабильного доступа к набору подов (по селектору labels).

| Тип | Назначение |
|-----|------------|
| **ClusterIP** | Виртуальный IP (из диапазона service-cluster-ip-range), доступный только внутри кластера. DNS-имя `<svc>.<ns>.svc.cluster.local` резолвится в этот IP. kube-proxy или CNI делают DNAT с ClusterIP на IP одного из подов. |
| **NodePort** | Порт (30000–32767) открыт на **каждом** узле; трафик на `<NodeIP>:<NodePort>` перенаправляется на ClusterIP и далее на под. Доступ снаружи кластера без внешнего балансировщика. |
| **LoadBalancer** | Обычно расширение NodePort: облачный контроллер создаёт внешний балансировщик (ALB/NLB) и выдаёт внешний IP. Трафик идёт через балансировщик → NodePort → Service → под. |

Вне кластера под по его Pod IP напрямую недоступен (сеть подов часто приватная); доступ — через Service (NodePort/LoadBalancer) или Ingress.

### kube-proxy (iptables / ipvs)

**kube-proxy** — компонент на каждой ноде, который поддерживает правила для Service: при обращении к ClusterIP:port трафик должен уйти на один из эндпоинтов (под). Режимы:

- **iptables** — правила в iptables (цепочки KUBE-SERVICES, KUBE-SEP-…). Просто, но при большом числе сервисов/подов таблица растёт.
- **ipvs** — использование ядерного IPVS (LVS): таблицы по типу hash и др., лучше масштабируется при тысячах сервисов.

kube-proxy не выделяет Pod IP; за подсеть подов отвечает **CNI**.

---

## CNI (Container Network Interface)

**CNI** — стандарт плагинов, которые при создании/удалении пода настраивают сеть: создают veth, выдают IP из пула, прописывают маршруты. Без CNI поды не получают IP и не видят друг друга. Разные плагины дают разную топологию, политики и производительность.

### Calico

**Calico** — CNI с поддержкой L3 (BGP или overlay), встроенные **NetworkPolicy** (в т.ч. eBPF), возможность работы без overlay (pure L3). Подходит для требований к производительности и к сетевой политике в enterprise.

### Cilium

**Cilium** — CNI на **eBPF**: маршрутизация и политики в ядре без iptables. Поддержка NetworkPolicy, observability (Hubble), L7-политики для HTTP. Часто используется в современных кластерах с акцентом на безопасность и видимость трафика.

### Flannel

**Flannel** — простой CNI: overlay-сеть (VXLAN или host-gw). Каждый под получает IP из выделенной подсети; маршруты между нодами настраиваются автоматически. Нет встроенных NetworkPolicy (их реализует kube-proxy или другой компонент). Удобен для быстрого старта и простых топологий.

---

## Ingress

### Ingress Controller

**Ingress** — ресурс Kubernetes, описывающий правила L7-маршрутизации: host, path → backend Service. Обрабатывает **Ingress Controller** (Nginx Ingress, HAProxy Ingress, Traefik, Istio Gateway и др.): он смотрит на объекты Ingress и настраивает прокси/балансировщик. Обычно Controller работает как под с Service типа LoadBalancer или NodePort и принимает трафик на 80/443.

### L7 routing

Правила Ingress задают: по какому **Host** и **path** запрос идёт в какой Service (и порт). Один внешний адрес/порт — много приложений. Поддержка TLS (сертификаты в Secret или через cert-manager). Это и есть L7-маршрутизация в Kubernetes без отдельного внешнего балансировщика с правилами по URL.

---

## NetworkPolicy

**NetworkPolicy** — ресурс Kubernetes для ограничения трафика между подами (и с/на внешний мир): какие поды могут общаться с какими по какому порту и протоколу. Реализуется плагином CNI (Calico, Cilium и др.) или компонентом, который переводит Policy в правила iptables/eBPF.

### Allow / Deny

Политика описывает **selector** (какие поды затронуты), **ingress** (кто может к ним подключаться) и **egress** (куда они могут подключаться). Правила — whitelist: разрешено только то, что явно указано (при включённой политике для namespace/подов).

### Default deny

Если для namespace или подов включена NetworkPolicy и нет правил, разрешающих трафик, по умолчанию он **запрещён** (default deny). Чтобы разрешить доступ, нужна политика с соответствующим ingress/egress. Многие проблемы «вдруг перестало ходить» после включения политик связаны с тем, что не описан нужный allow.

!!! warning "Внимание"

    NetworkPolicy поддерживается не всеми CNI. Flannel «из коробки» не реализует NetworkPolicy; нужен Calico, Cilium или другой плагин с поддержкой. Без такого плагина создание NetworkPolicy не даёт эффекта.

---

## Как под из namespace A ходит в под из namespace B?

1. **По Pod IP:** если известен IP пода в namespace B, под в A может обратиться напрямую по этому IP (если CNI обеспечивает связность между нодами и нет блокировки NetworkPolicy). Подходит для разовых проверок, но IP нестабилен.
2. **По Service:** стабильный способ — Service в namespace B. Из пода в A обращаются по DNS: `<service>.<namespace-b>.svc.cluster.local` или коротко `<service>.<namespace-b>` (если search в resolv.conf включает нужный namespace). DNS резолвит в ClusterIP; kube-proxy/CNI делают DNAT на один из подов Service в B.
3. **Сеть:** CNI выдаёт подам IP из общей подсети (или маршрутизируемых подсетей); маршруты на нодах настроены так, что пакет с ноды A доходит до ноды B и попадает в network namespace пода. Firewall и NetworkPolicy не должны блокировать этот трафик.

Итого: типичный способ — **Service в B + DNS-имя**; под капсулой — Pod IP и маршрутизация CNI.

---

## Почему NetworkPolicy не работает?

Типичные причины:

1. **CNI не поддерживает NetworkPolicy** — например, «голой» Flannel. Политика создаётся, но правил в ядре нет. Нужен плагин с поддержкой (Calico, Cilium и т.д.).
2. **Default deny без allow** — включена изоляция, но не добавлено правило, разрешающее нужный трафик. Нужно явно описать ingress/egress для подов и портов.
3. **Селекторы не совпадают** — podSelector в Policy не выбирает нужные поды (другие labels) или не указан namespace. Проверить labels подов и namespace в Policy.
4. **Порты/протокол не указаны** — в правиле указан только podSelector, но не port; часть трафика может блокироваться в зависимости от реализации. Явно указывать ports в Policy.
5. **Политика применяется не к тем подам** — policySelector или namespace в Policy не охватывает целевые поды. Убедиться, что Policy в нужном namespace и селекторы верные.

Диагностика: проверить, что CNI поддерживает NetworkPolicy; посмотреть правила в iptables/eBPF на ноде (Calico/Cilium имеют свои команды); проверить labels и наличие разрешающих правил.

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Доступ к подам через Service** | Стабильное имя и IP; не полагаться на Pod IP в приложениях. |
| **Знать свой CNI** | От него зависят Pod network, поддержка NetworkPolicy и способ отладки. |
| **Сначала разрешить, потом ограничивать** | При вводе NetworkPolicy начинать с явных allow, потом при необходимости default deny. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Ждать NetworkPolicy на Flannel без доп. плагина** | Правила не применяются. | Использовать CNI с поддержкой NetworkPolicy или не полагаться на политики. |
| **Default deny без теста** | Сразу режется весь трафик, включая DNS и системные компоненты. | Добавлять allow для DNS (CoreDNS), kube-system и приложений по шагам и проверять связность. |

---

## Дополнительные материалы

- [Kubernetes — Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Kubernetes — Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes — Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes — NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [CNI — spec](https://github.com/containernetworking/cni/blob/master/SPEC.md)
- [Calico](https://www.tigera.io/project-calico/)
- [Cilium](https://cilium.io/)
- [Flannel](https://github.com/flannel-io/flannel)
