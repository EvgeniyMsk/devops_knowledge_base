# 4. Сеть в Kubernetes

---
Pod-to-Pod, Service и kube-proxy (iptables/ipvs), DNS (CoreDNS), Ingress и TLS (Cert-Manager), Ingress Controllers, Gateway API, CNI (Calico/Cilium/Flannel), Service Mesh (Istio, Linkerd: mTLS, маршрутизация, observability) + практика Ingress/TLS. Примеры манифестов с комментариями и best practices для production.

---

## Термины и сущности сети

| Термин | Определение |
|--------|-------------|
| **Pod-to-Pod** | Прямая сетевая связь между подами по их IP из подсети CNI; нестабильна при пересоздании пода, для стабильного доступа используют Service. |
| **ClusterIP** | Тип Service: виртуальный IP внутри кластера; трафик к ClusterIP:port перенаправляется kube-proxy на один из эндпоинтов (под). |
| **NodePort** | Тип Service: порт в диапазоне 30000–32767 на каждой ноде; трафик с узла идёт в ClusterIP и далее в под. |
| **LoadBalancer** | Тип Service: облачный контроллер создаёт внешний балансировщик и выдаёт внешний IP; обычно расширение NodePort. |
| **Headless Service** | Service с `clusterIP: None`; DNS возвращает список IP подов (для StatefulSet и прямого доступа к подам). |
| **kube-proxy** | Компонент на каждой ноде: поддерживает правила iptables/ipvs для Service (ClusterIP, NodePort); L4, не L7. |
| **CoreDNS** | Стандартный DNS-сервер в кластере; резолвит имена сервисов и подов (*.svc.cluster.local, *.pod.cluster.local). |
| **CNI (Container Network Interface)** | Плагин, настраивающий сеть для подов: выделение IP, маршруты, overlay; без CNI поды не получают IP. |
| **Ingress** | Объект API для L7-маршрутизации: по Host и path трафик направляется на нужный Service; обрабатывается Ingress Controller. |
| **Ingress Controller** | Компонент (NGINX, Traefik и др.), который читает объекты Ingress и настраивает прокси/балансировщик; без него Ingress не работает. |
| **Cert-Manager** | Контроллер в кластере: выдаёт и продлевает TLS-сертификаты (например, Let's Encrypt), хранит их в Secret. |
| **Gateway API** | Современный API для L4/L7: разделение Gateway (точка входа) и HTTPRoute/TCPRoute (маршруты); замена/развитие Ingress. |
| **NetworkPolicy** | Объект API: правила входящего и исходящего трафика для подов (L3/L4); реализуется CNI-плагином (Calico, Cilium и др.). |
| **Service Mesh** | Слой управления трафиком между сервисами: sidecar-прокси на каждом поде, mTLS, маршрутизация, observability; примеры: Istio, Linkerd. |

---

## Основы: Pod-to-Pod и Service

### Pod-to-Pod Networking

Поды получают IP из подсети CNI; в пределах кластера они могут общаться друг с другом по этим IP. Маршруты на нодах и правила CNI обеспечивают доставку пакетов между нодами. Прямая связь по Pod IP нестабильна (под может пересоздаться); для стабильного доступа используют **Service** и DNS.

Типы Service (ClusterIP, NodePort, LoadBalancer) и Headless рассмотрены в [3. Workloads и API-объекты](topic-3-workloads-api.md). Кратко: **ClusterIP** — виртуальный IP внутри кластера; **NodePort** — порт на каждой ноде; **LoadBalancer** — внешний балансировщик в облаке. **kube-proxy** на каждой ноде поддерживает правила (iptables или ipvs): трафик к ClusterIP:port перенаправляется на один из эндпоинтов (под). Режим **ipvs** лучше масштабируется при большом числе сервисов; режим задаётся при установке (например, флаг kube-proxy).

---

## DNS в Kubernetes (CoreDNS)

**CoreDNS** — стандартный DNS-сервер в кластере. Резолвит:

- Имена сервисов: `<service>.<namespace>.svc.cluster.local` → ClusterIP (или список IP для Headless).
- Короткие имена: `<service>` или `<service>.<namespace>` — через search-список в `/etc/resolv.conf` пода (обычно `namespace.svc.cluster.local`, `svc.cluster.local`, `cluster.local`).

Проверка из пода: `nslookup myapp-svc`, `dig myapp-svc.default.svc.cluster.local`. При проблемах с «не резолвится имя» смотреть: под в том же namespace, что и Service; CoreDNS поды работают; нет ли блокировки NetworkPolicy.

---

## Ingress

**Ingress** — объект Kubernetes для L7-маршрутизации: по Host и path трафик направляется на нужный Service. Обрабатывает **Ingress Controller** (NGINX, Traefik и др.): он смотрит на ресурсы Ingress и настраивает прокси/балансировщик. TLS можно терминировать на Ingress (сертификаты в Secret или через Cert-Manager). Без Ingress Controller объекты Ingress сами по себе ничего не делают — нужна установка контроллера в кластер.

---

## Примеры манифестов (с комментариями)

### Ingress: один Host, path-based routing

```yaml
# Ingress направляет трафик по путям на разные сервисы.
# Требуется установленный Ingress Controller (например, NGINX).
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: default
  annotations:
    # NGINX: увеличить размер буфера для больших заголовков (например, JWT)
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
    # Таймаут бэкенда (по умолчанию 60s)
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
  # TLS: секрет с сертификатом (см. Cert-Manager ниже)
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
```

### Ingress: несколько Host и TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
        - www.example.com
      secretName: example-com-wildcard-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 8080
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

### Cert-Manager: ClusterIssuer (Let’s Encrypt)

```yaml
# ClusterIssuer — доступен во всех namespace.
# Используется для выдачи сертификатов через Let's Encrypt (ACME).
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      # HTTP-01: проверка через HTTP по пути .well-known/acme-challenge
      - http01:
          ingress:
            class: nginx
```

### Cert-Manager: Issuer (в рамках namespace)

```yaml
# Issuer действует только в своём namespace.
# Удобно для разных окружений (dev/stage/prod) с разными CA.
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: default
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

### Cert-Manager: Certificate (запрос сертификата)

```yaml
# Certificate создаёт запрос на сертификат; cert-manager обращается к Issuer/ClusterIssuer
# и кладёт выданный сертификат в Secret. Ingress ссылается на этот Secret в tls[].secretName.
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-example-com-tls
  namespace: default
spec:
  secretName: app-example-com-tls   # куда сохранить сертификат (для Ingress)
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - app.example.com
    - www.example.com
```

### Ingress с аннотациями NGINX: rewrite и rate limit (production-ориентировано)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Rewrite: убрать префикс /v1 перед передачей в бэкенд
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    # Ограничение частоты запросов (защита от простого DDoS/злоупотреблений)
    nginx.ingress.kubernetes.io/limit-rps: "100"
    # Включить CORS при необходимости (пример)
    # nginx.ingress.kubernetes.io/enable-cors: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
  rules:
    - host: app.example.com
      http:
        paths:
          # Запросы на /v1/... уйдут в api-svc на путь /...
          - path: /v1(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

## Gateway API

**Gateway API** — современная замена и развитие идеи Ingress: стандартизированный, ролеориентированный API для L7-маршрутизации (и при необходимости L4). Разрабатывается в рамках SIG Network; поддерживается рядом контроллеров (NGINX Gateway Fabric, Envoy Gateway, Istio, Cilium и др.). В отличие от одного ресурса Ingress, Gateway API разделяет **инфраструктуру** (кто предоставляет точку входа) и **маршруты** (куда направлять трафик), что удобно для мультитенантности и разделения прав между платформой и приложениями.

### Gateway API vs Ingress

| Аспект | Ingress | Gateway API |
|--------|---------|-------------|
| **Модель** | Один объект: и точка входа, и правила в одном месте. | Разделение: Gateway (инфра) + HTTPRoute/TCPRoute и т.д. (маршруты). |
| **Роли** | Один набор прав на объект. | Gateway — часто оператор/платформа; HTTPRoute — команда приложения в своём namespace. |
| **Расширяемость** | Аннотации (разные у каждого контроллера). | Встроенные фильтры (redirect, rewrite, header), расширение через политики (future). |
| **Протоколы** | В основном HTTP/HTTPS. | HTTP, HTTPS, gRPC (GRPCRoute), TCP (TCPRoute), UDP. |
| **Статус** | Стабильный, но ограниченный. | Активная разработка; часть ресурсов уже GA в Kubernetes 1.29+. |

Для новых проектов и мультитенантных кластеров имеет смысл оценивать Gateway API; для существующих Ingress-контроллеров миграция возможна пошагово (некоторые контроллеры поддерживают и Ingress, и Gateway API одновременно).

### Основные ресурсы

- **GatewayClass** — шаблон типа шлюза (какой контроллер обрабатывает). Администратор или оператор создаёт класс (например, `nginx`, `envoy`); в кластере может быть несколько классов.
- **Gateway** — экземпляр точки входа: слушатели (listeners) по портам и протоколам, TLS. Обычно создаётся оператором или через контроллер по заявке; приложение затем привязывает к нему маршруты через **parentRefs**.
- **HTTPRoute** — правила маршрутизации HTTP/HTTPS: привязка к Gateway (`parentRefs`), hostnames, правила (matches по path/header + backendRefs на Service). Аналог правил из Ingress, но с явной привязкой к шлюзу и богаче возможностями (фильтры, веса бэкендов).
- **GRPCRoute**, **TCPRoute**, **UDPRoute** — маршруты для других протоколов (зависит от реализации контроллера).

### GatewayClass

```yaml
# GatewayClass определяет «тип» шлюза; контроллер с этим controllerName обрабатывает Gateway с данным классом.
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
  description: "NGINX Gateway Fabric"
```

Имя класса (например, `nginx`) затем указывается в поле **gatewayClassName** у Gateway.

### Gateway: listeners и TLS

```yaml
# Gateway — точка входа. Создаётся оператором или через Helm; в status контроллер пишет адреса (LoadBalancer IP и т.д.).
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: example-com-tls
            namespace: default
      allowedRoutes:
        namespaces:
          from: All
```

- **listeners** — порты и протоколы (HTTP, HTTPS, при поддержке — TLS passthrough).
- **tls.mode: Terminate** — TLS терминируется на шлюзе; **certificateRefs** указывают на Secret с сертификатом (тот же формат, что и для Ingress; можно выдавать через Cert-Manager).
- **allowedRoutes** — откуда разрешено привязывать маршруты (те же namespace, все namespace, или по селектору). Это даёт возможность изолировать, какой команде какой Gateway доступен.

Проверка: после создания Gateway контроллер заполняет **status.addresses** и **status.listeners**; `kubectl get gateway` и `kubectl describe gateway` покажут готовность.

### HTTPRoute: правила и привязка к Gateway

```yaml
# HTTPRoute привязывается к Gateway через parentRefs; hostnames и rules задают маршрутизацию (аналог Ingress rules).
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: default
spec:
  parentRefs:
    - name: public-gateway
      namespace: default
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-svc
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: web-svc
          port: 80
```

- **parentRefs** — к какому Gateway (и при необходимости к какому listener) привязана маршрутизация.
- **hostnames** — аналог `host` в Ingress; можно несколько.
- **rules** — список правил; каждое правило: **matches** (path, headers, method) и **backendRefs** (Service в том же или другом namespace). При совпадении запроса трафик уходит на указанные бэкенды.

### HTTPRoute: фильтры (redirect, rewrite, headers)

В **rules** можно задавать **filters** — модификация запроса/ответа или редирект до отправки в бэкенд. Поддержка зависит от контроллера.

```yaml
# Пример: редирект с HTTP на HTTPS и добавление заголовка перед отправкой в бэкенд.
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route-with-filters
  namespace: default
spec:
  parentRefs:
    - name: public-gateway
      namespace: default
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: X-Request-Source
                value: gateway
      backendRefs:
        - name: web-svc
          port: 80
---
# Редирект (обычно на уровне listener или отдельного rule).
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: redirect-http-to-https
  namespace: default
spec:
  parentRefs:
    - name: public-gateway
      namespace: default
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
      backendRefs: []
```

Типы фильтров (по спецификации): **RequestHeaderModifier**, **ResponseHeaderModifier**, **RequestRedirect**, **URLRewrite**, **ExtensionRef** (контроллер-специфичные). Конкретный синтаксис и поддержку смотри в документации выбранного контроллера.

### HTTPRoute: взвешенное распределение (canary)

Можно указать несколько **backendRefs** с разными **weight** — часть трафика на один сервис, часть на другой (canary / A/B).

```yaml
# 90% трафика на stable-svc, 10% на canary-svc.
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-canary-route
  namespace: default
spec:
  parentRefs:
    - name: public-gateway
      namespace: default
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: stable-svc
          port: 80
          weight: 90
        - name: canary-svc
          port: 80
          weight: 10
```

### Статус и отладка

Контроллер заполняет **status** у Gateway и HTTPRoute: готовность listener’ов, условия (Accepted, ResolvedRefs и т.д.). При проблемах с маршрутизацией смотреть:

```bash
kubectl get gateway
kubectl describe gateway public-gateway
kubectl get httproute
kubectl describe httproute app-route
```

В **status** не должно быть False в важных условиях; если parentRef не найден или backendRef недоступен, в status будет указана причина.

### TLS с Gateway API и Cert-Manager

Сертификаты хранятся в Secret; Gateway ссылается на них через **certificateRefs**. Выдавать сертификаты можно так же, как для Ingress: **Certificate** (Cert-Manager) с нужным secretName; после выдачи Secret создаётся, и Gateway его подхватывает. Имя Secret и namespace в **certificateRefs** должны совпадать с тем, куда Cert-Manager пишет сертификат.

### Best practices для Gateway API

- **Разделение ролей:** оператор создаёт GatewayClass и экземпляры Gateway; команды приложений создают только HTTPRoute в своих namespace, привязывая к общему Gateway через parentRefs. Ограничивать **allowedRoutes** по namespace при необходимости.
- **Один Gateway на окружение/назначение** — не плодить лишние Gateway; маршруты разделять по hostnames и HTTPRoute.
- **Проверять status** после создания Gateway и HTTPRoute; при смене бэкенда убедиться, что ResolvedRefs в порядке (Service существует и порт корректен).
- **TLS:** по возможности использовать Cert-Manager и не хранить сертификаты вручную; формат Secret для Gateway такой же, как для Ingress (tls.crt, tls.key).

---

## Ingress Controllers: NGINX и Traefik

- **NGINX Ingress Controller** — один из самых распространённых. Устанавливается через Helm или манифесты; создаёт под(ы) и Service (часто LoadBalancer или NodePort). Класс Ingress по умолчанию — `nginx`. Аннотации `nginx.ingress.kubernetes.io/*` задают rewrite, rate limit, CORS, буферы и т.д.
- **Traefik** — альтернатива с динамической конфигурацией и встроенной панелью. Использует CRD (IngressRoute) или стандартный Ingress; аннотации и класс другие (`traefik`). Подходит для тех, кто предпочитает один инструмент для ingress и внутренней маршрутизации.

Выбор зависит от знакомства команды и требований (rewrite, WAF, метрики). В production важно зафиксировать версию контроллера и тестировать обновления.

---

## CNI: Calico, Cilium, Flannel

- **Calico** — L3/L2, поддержка NetworkPolicy, опционально BGP. Подходит для строгих требований к политикам и производительности без overlay.
- **Cilium** — на базе eBPF: маршрутизация и политики в ядре, observability (Hubble). Сильная сторона — безопасность и видимость трафика, L7-политики для HTTP.
- **Flannel** — простой overlay (VXLAN или host-gw). NetworkPolicy не встроены (реализуются через другой компонент). Удобен для быстрого старта и простых топологий.

Подробнее — в [Networking: 8. Kubernetes Networking](../../networking/topic-8-kubernetes-networking.md). В production выбор CNI влияет на NetworkPolicy, метрики и отладку сети.

---

## NetworkPolicy: сегментация трафика внутри кластера

**NetworkPolicy** — объект Kubernetes, который описывает, какой входящий и исходящий трафик разрешён для подов (L3/L4: IP/порт/протокол). Работает только если CNI-плагин поддерживает политики (Calico, Cilium и др.). Без NetworkPolicy поды в namespace, как правило, могут общаться со всеми подами/сервисами во всех namespace.

Ключевые идеи:

- Политика действует **на выбранные поды** (selector), а не на весь кластер.
- Правила делятся на **ingress** (входящий трафик) и **egress** (исходящий).
- Модель «deny by default» достигается тем, что при наличии хотя бы одной политики для пода **весь неописанный трафик запрещается**.
- Политики **аддитивны** — если несколько NetworkPolicy матчатся на один под, разрешения суммируются.

### Базовый пример: запрет всего входящего кроме namespace

Разрешим входящий трафик к подам `app=backend` только из подов в том же namespace с меткой `app=frontend`. Всё остальное входящее будет запрещено.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

- **podSelector** — какие поды защищаем (здесь все `app=backend` в namespace `prod`).
- **policyTypes: [Ingress]** — управляем только входящим трафиком.
- **ingress[].from[].podSelector** — кто может подключаться (подов `app=frontend` из того же namespace).
- **ports** — на какие порты разрешён трафик (8080/TCP).

После применения: к `backend` никто, кроме `frontend` в `prod`, войти не сможет (при условии, что CNI поддерживает NetworkPolicy).

### Разрешить только нужные egress-направления

Ограничим исходящий трафик подов с меткой `role=app` только на базу данных и DNS.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-egress-db-dns
  namespace: prod
spec:
  podSelector:
    matchLabels:
      role: app
  policyTypes:
    - Egress
  egress:
    # Разрешить доступ к БД
    - to:
        - namespaceSelector:
            matchLabels:
              name: prod-db
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Разрешить доступ к DNS (CoreDNS)
    - to:
        - namespaceSelector:
            matchLabels:
              kube-system: "true"
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

- **policyTypes: [Egress]** — управляем только исходящим трафиком.
- Первая запись в **egress** позволяет ходить в namespace `prod-db` к подам `app=postgres` на порт 5432/TCP.
- Вторая — разрешает DNS-запросы к CoreDNS (порт 53/UDP в namespace `kube-system`).
- Любой другой исходящий трафик от подов `role=app` будет запрещён.

### Политика «deny all» для namespace

Частый production-подход — сделать namespace «закрытым по умолчанию» и добавлять точечные allow-политики.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

- **podSelector: {}** — все поды в namespace.
- **Ingress + Egress** — запрещаем весь входящий и исходящий трафик по умолчанию.
- После этого нужно добавить отдельные NetworkPolicy с ingress/egress-правилами для конкретных ролей (`frontend`, `backend`, `db` и т.д.).

### Разрешить трафик из других namespace по меткам

Иногда нужно разрешить доступ из инфраструктурных namespace (например, мониторинг).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9100
```

- **namespaceSelector** — выбираем namespace по меткам (здесь `name=monitoring`).
- Разрешаем входящий трафик на порт 9100/TCP (обычный порт для метрик) только из namespace мониторинга.

### Отладка и проверка NetworkPolicy

Полезные команды:

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy -n prod default-deny-all
```

Лучший способ проверки — запуск тестового пода без ограничений и попытка подключиться:

```bash
kubectl run -n prod netshoot --image=nicolaka/netshoot -it --rm -- bash
# Внутри:
curl -v http://backend:8080/health
nc -vz postgres.prod-db.svc.cluster.local 5432
```

Если доступ неожиданно закрыт или открыт — пересмотреть правила и пересобрать политики (особенно важно помнить, что политики аддитивны).

### Best practices для NetworkPolicy

- **Deny by default для критичных namespace**: `default-deny-all` или хотя бы `default-deny-ingress`, затем точечные allow-политики.
- **Использовать метки, а не IP**: селекторы по labels (`app`, `role`, `tier`) вместо IP/CIDR — проще сопровождать при масштабировании.
- **Разделять ingress и egress**: чётко понимать, кто может приходить к сервису, и куда сервис имеет право ходить.
- **Документировать групповыми политиками**: хранить политики вместе с манифестами приложения (GitOps), разбивать по ролям (`frontend`, `backend`, `db`).
- **Тестировать на staging**: сначала включать строгие политики в non-prod кластере, проверять сценарии миграции/обновлений.
- **Не забывать про DNS и внешние интеграции**: явно разрешать egress к DNS, шлюзам платежей, внешним API и т.п.

---

## Service Mesh

**Service Mesh** — слой управления трафиком между сервисами (service-to-service) внутри кластера. Трафик перехватывается **sidecar-прокси** (прокси-контейнер рядом с каждым подом приложения); **control plane** выдаёт конфигурацию прокси и собирает телеметрию. Приложение не меняет код: шифрование (mTLS), retry, таймауты, метрики и маршрутизация (canary, A/B) настраиваются через CRD или API mesh’а.

Зачем нужен Service Mesh:

- **mTLS (mutual TLS)** — шифрование и взаимная аутентификация между подами без изменений в коде приложения.
- **Observability** — метрики, трейсы и логи на уровне запросов между сервисами (latency, error rate, topology).
- **Traffic management** — canary, A/B, weighted routing, retry/timeout без выноса логики в Ingress или код.
- **Resilience** — circuit breaker, outlier detection (исключение «плохих» эндпоинтов из балансировки).

Основные реализации в экосистеме Kubernetes: **Istio** (Envoy sidecar, богатый набор CRD, интеграция с экосистемой), **Linkerd** (лёгкий Rust-прокси, простота установки и эксплуатации), **Cilium Service Mesh** (на базе eBPF, без sidecar). Ниже — концепции и примеры на Istio; принципы применимы и к другим mesh’ам.

### Концепции: Data Plane и Control Plane

- **Data plane** — набор sidecar-прокси (в Istio — Envoy). Каждый под приложения получает дополнительный контейнер (inject); исходящий и входящий трафик пода идёт через этот прокси. Прокси получает конфигурацию от control plane и отдаёт метрики/трейсы.
- **Control plane** — компоненты mesh’а (в Istio: istiod). Выдаёт конфигурацию прокси (например, куда слать трафик, какие retry/таймауты), управляет сертификатами для mTLS, агрегирует телеметрию. В кластере обычно один control plane (с репликой для HA).

Включение в mesh: помечают namespace меткой (например, `istio-injection=enabled`) или аннотацией у пода; при создании пода **injector** добавляет sidecar-контейнер. Трафик между подами в mesh идёт через sidecar и может автоматически шифроваться (mTLS).

### Istio: установка и включение sidecar

Установка Istio (кратко): через `istioctl install` или Helm; выбирается профиль (default, demo, minimal и т.д.). Включение инъекции sidecar для namespace:

```yaml
# Включить автоматическую инъекцию sidecar для namespace.
# Поды, создаваемые в этом namespace, получат контейнер istio-proxy (Envoy).
apiVersion: v1
kind: Namespace
metadata:
  name: app
  labels:
    istio-injection: enabled
```

Проверка: после создания пода в `app` у пода должно быть два контейнера (приложение + istio-proxy). Запросы из пода к другому сервису в mesh идут через sidecar и учитываются в метриках Istio.

### VirtualService и DestinationRule: маршрутизация и политики подбора

**VirtualService** задаёт правила маршрутизации к сервису (по заголовкам, URI, весам). **DestinationRule** описывает подмножества (subsets) бэкенда и политики (load balancing, connection pool, outlier detection).

Пример: canary по весам и подмножествам по версии.

```yaml
# DestinationRule: определяем подмножества (v1, v2) по метке версии.
# Политики: round-robin, outlier detection — выкидывать эндпоинт из балансировки при ошибках.
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-dr
  namespace: default
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---
# VirtualService: 90% трафика на v1, 10% на v2 (canary).
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-vs
  namespace: default
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: internal
      route:
        - destination:
            host: reviews
            subset: v2
        weight: 100
    - route:
        - destination:
            host: reviews
            subset: v1
        weight: 90
        - destination:
            host: reviews
            subset: v2
        weight: 10
```

- **DestinationRule.host** — имя сервиса Kubernetes (reviews). **subsets** — выбор подов по labels; **trafficPolicy** применяется ко всему трафику к этому host (можно переопределять в subset).
- **VirtualService.hosts** — тот же сервис. **http[].match** — условия (headers, uri, method); **route** — куда вести трафик и с каким weight. Здесь: для заголовка `end-user: internal` — весь трафик на v2; иначе — 90% v1, 10% v2.

### Retry и timeout в VirtualService

Устойчивость к временным сбоям и ограничение времени ожидания задаются в том же **http** rule:

```yaml
# Retry при 5xx и timeout 3s для вызовов к ratings.
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-vs
  namespace: default
spec:
  hosts:
    - ratings
  http:
    - route:
        - destination:
            host: ratings
      timeout: 3s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: 5xx,reset,connect-failure
```

- **timeout** — общий таймаут запроса со стороны вызывающего.
- **retries.attempts** — число попыток; **perTryTimeout** — таймаут одной попытки; **retryOn** — при каких условиях повторять (5xx, сброс соединения и т.д.). В production не увеличивать attempts без ограничения, чтобы не усугублять нагрузку на сломанный бэкенд.

### mTLS: PeerAuthentication

Включение взаимной TLS между подами в mesh — через **PeerAuthentication**. По умолчанию Istio может работать в режиме PERMISSIVE (принимает и plaintext, и mTLS); для строгого шифрования между namespace задают STRICT.

```yaml
# Включить строгий mTLS для всего namespace default:
# только зашифрованные соединения между подами.
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-mtls
  namespace: default
spec:
  mtls:
    mode: STRICT
---
# Глобальная политика для всего mesh (обычно в namespace istio-system).
# apiVersion: security.istio.io/v1beta1
# kind: PeerAuthentication
# metadata:
#   name: default
#   namespace: istio-system
# spec:
#   mtls:
#     mode: STRICT
```

- **STRICT** — sidecar принимает только mTLS; исходящие вызовы из этого namespace тоже идут по mTLS (если целевой сервис в mesh).
- **PERMISSIVE** — принимаются и mTLS, и plaintext (удобно при постепенном включении mesh).
- **DISABLE** — mTLS отключён для этого scope (namespace/workload). Использовать точечно, например для legacy-подов без sidecar.

Сертификаты для mTLS в Istio по умолчанию выдаёт control plane (istiod); продление автоматическое. В production при необходимости можно перейти на собственный CA (custom CA в конфиге Istio).

### AuthorizationPolicy: кто может вызывать сервис

Разрешить или запретить вызовы по принципу «кто звонит» (identity из mTLS или из атрибутов запроса):

```yaml
# Разрешить вызовы к reviews только от сервиса productpage (на основе mTLS identity).
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: reviews-allow-productpage
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/productpage"]
```

- **selector** — к каким подам применяется политика (здесь — с label app=reviews).
- **action: ALLOW** — разрешаются только запросы, подходящие под **rules**; остальные по умолчанию запрещены (если включён authorization). **principals** — identity сервисного аккаунта вызывающего (формат Istio: cluster/ns/namespace/sa/serviceaccount).

Можно комбинировать **from** (source) и **to** (method, path) для детальных правил. В production начинать с явного ALLOW для нужных источников и не открывать «все» без необходимости.

### Пример: Linkerd (кратко)

Linkerd использует свой лёгкий прокси (Rust), без Envoy. Концепции похожи: инъекция sidecar, mTLS из коробки, метрики (golden metrics: latency, success rate, throughput). Маршрутизация по весам и retry настраиваются через **ServiceProfile** (CRD Linkerd). Установка: `linkerd inject` для манифестов или аннотация namespace. Плюсы — низкое потребление ресурсов и простая установка; минусы — меньше возможностей по сравнению с Istio (например, по сложной маршрутизации по заголовкам).

### Best practices для Service Mesh в production

| Практика | Описание |
|----------|----------|
| **Включать mesh по namespace** | Включать инъекцию по одному namespace/приложению, проверять метрики и mTLS, затем расширять. |
| **Задавать лимиты для sidecar** | У sidecar (istio-proxy) задавать requests/limits CPU и memory, чтобы не конкурировать с приложением при нехватке ресурсов. |
| **Retry с осторожностью** | Ограничивать attempts и использовать retryOn (например, только 5xx), чтобы не усиливать каскадные сбои. |
| **DestinationRule с outlier detection** | Включать outlier detection для автоматического исключения «падающих» эндпоинтов из балансировки. |
| **mTLS по умолчанию STRICT** | После выката mesh переходить на STRICT в нужных namespace; оставлять PERMISSIVE только для миграции. |
| **Мониторинг control plane** | Следить за состоянием istiod/linkerd control plane и за количеством сбоев конфигурации (например, в Prometheus). |
| **Не мешать readiness** | Убедиться, что readiness probe приложения не идёт через sidecar туда, где это меняет результат; при необходимости проверять приложение напрямую. |

### Паттерны и антипаттерны (Service Mesh)

| Паттерн | Описание |
|--------|----------|
| **Canary через VirtualService** | Выкат новой версии как отдельный subset; дать 5–10% трафика, смотреть метрики, затем увеличивать weight. |
| **Один DestinationRule на сервис** | Хранить subsets и outlier detection в одном DR; ссылаться на subsets из нескольких VirtualService при необходимости. |
| **AuthorizationPolicy по принципу least privilege** | Явно разрешать только нужных вызывающих (по principal); не полагаться на «все разрешено». |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Включать mesh сразу на весь кластер** | Сбои инъекции или control plane затронут все приложения. | Включать по namespace/приложению, проверять стабильность. |
| **Без лимитов у sidecar** | При нехватке памяти/CPU sidecar может «задавить» приложение. | Задавать requests/limits для контейнера istio-proxy/linkerd-proxy. |
| **Бесконечные retry** | Увеличивает нагрузку на сломанный бэкенд и задержки для пользователя. | Ограничивать attempts (2–3), задавать perTryTimeout и retryOn. |
| **Игнорировать телеметрию mesh** | Теряется смысл mesh — видимость и управление трафиком. | Интегрировать метрики Istio/Linkerd с Prometheus/Grafana и настроить алерты. |

---

## Best practices и примеры из production

### TLS Termination

- **Терминировать TLS на Ingress**, а не в каждом поде: один сертификат и один пункт настройки. Бэкенд-сервисы за Ingress работают по HTTP (внутри кластера).
- Использовать **Cert-Manager** и **ClusterIssuer/Issuer** для автоматического продления сертификатов (Let’s Encrypt). Не хранить долгоживущие сертификаты вручную в Secret без продления.
- В production: **TLS 1.2 минимум**, отключение слабых шифров в Ingress Controller (аннотации или конфиг контроллера).

### Path-based routing и rewrite

- Чётко разделять пути по сервисам: `/api` → api-svc, `/` → frontend. Использовать `pathType: Prefix` или `ImplementationSpecific` в зависимости от контроллера.
- **Rewrite** (например, убрать префикс `/v1` перед бэкендом) настраивать аннотациями контроллера (`rewrite-target` в NGINX). Тестировать с curl с хоста и из пода.
- Избегать дублирования правил: порядок правил в Ingress важен (часто первое совпадение побеждает). Более специфичные пути — выше в списке.

### Ограничения и безопасность

- **Rate limiting** на Ingress (limit-rps, limit-connections в NGINX) — защита от всплесков и простых атак. Значения подбирать по метрикам и SLA.
- **Размер буферов** (proxy-buffer-size) увеличивать при больших заголовках (JWT, cookies). Иначе 502 при длинных заголовках.
- **Таймауты** (proxy-connect-timeout, proxy-read-timeout) выставлять в соответствии с бэкендом. Долгие запросы (загрузка файлов, long polling) — увеличивать.

### Проверка доступности из пода

После настройки Ingress и Service проверять доступность с точки зрения приложения (из пода):

```bash
# Запустить под с curl (если образа нет — использовать image busybox и wget)
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl -v http://web-svc/default/
# По имени сервиса в том же namespace
kubectl exec -it <pod> -- curl -v http://api-svc:8080/health
# Через Ingress (если из пода резолвится внешний host)
kubectl exec -it <pod> -- curl -v -H "Host: app.example.com" http://ingress-controller-svc/
```

Убедиться, что DNS резолвит имена сервисов и что бэкенд отвечает (readiness probe и реальный endpoint).

---

## Практика: Ingress + TLS

1. Установить Ingress Controller (например, NGINX: `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.x/deploy/static/provider/cloud/deploy.yaml` или Helm).
2. Установить Cert-Manager (Helm или манифесты с официального сайта).
3. Создать ClusterIssuer для Let's Encrypt (staging для теста, затем prod).
4. Создать Certificate с нужными dnsNames и secretName.
5. Создать Ingress с правилами и `tls[].secretName` = secretName из Certificate.
6. Дождаться выдачи сертификата: `kubectl get certificate`, `kubectl describe certificate <name>`.
7. Проверить: `curl -v https://your-host/` и из пода: `kubectl exec -it <pod> -- curl -v http://<service>/`.

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **TLS на Ingress, HTTP к бэкенду** | Один пункт управления сертификатами и шифрованием. |
| **Cert-Manager для продления** | Не полагаться на ручное обновление сертификатов. |
| **Rate limit и таймауты** | Задавать на Ingress для защиты и предсказуемого поведения. |
| **Проверять с пода** | Доступность Service и Ingress с точки зрения приложения. |
| **Gateway API: разделение Gateway и HTTPRoute** | Оператор владеет точкой входа; приложения создают только маршруты в своих namespace. |
| **Service Mesh: canary и mTLS** | Маршрутизация по весам и шифрование между подами без изменений в коде приложения. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Хранить сертификаты вручную без продления** | Истечение срока — обрыв HTTPS. | Cert-Manager + Issuer/ClusterIssuer. |
| **Игнорировать status у Gateway/HTTPRoute** | Маршрут может не примениться (неверный parentRef, недоступный backend). | После создания проверять kubectl describe gateway/httproute и исправлять refs. |
| **Забыть про pathType и порядок правил** | Неверная маршрутизация, 404. | Явно задавать pathType, тестировать пути. |
| **Без rate limit на публичном Ingress** | Уязвимость к всплескам трафика. | Включить limit-rps/limit-connections. |

---

## Дополнительные материалы

- [Kubernetes — Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Cert-Manager](https://cert-manager.io/docs/)
- [Gateway API](https://gateway-api.sigs.k8s.io/) — обзор, концепции, [HTTPRoute](https://gateway-api.sigs.k8s.io/api-types/httproute/), [Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/)
- [Istio](https://istio.io/latest/docs/) — документация, [Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/), [Security](https://istio.io/latest/docs/concepts/security/)
- [Linkerd](https://linkerd.io/docs/) — документация, [Getting Started](https://linkerd.io/docs/getting-started/)
- [Networking: 8. Kubernetes Networking](../../networking/topic-8-kubernetes-networking.md)
- [Traefik Ingress](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/)
