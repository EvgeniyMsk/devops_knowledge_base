# 4. Сеть в Kubernetes

**Цель:** уверенно настраивать и отлаживать сеть в кластере: Pod-to-Pod, типы Service и kube-proxy (iptables/ipvs), DNS (CoreDNS), Ingress и TLS (Cert-Manager), Ingress Controllers (NGINX, Traefik), обзор Gateway API, CNI (Calico, Cilium, Flannel). Практика: Ingress + TLS, проверка доступности сервисов из пода. В разделе — примеры манифестов с комментариями и best practices для production.

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

Подробнее — в [Networking: 8. Kubernetes Networking](../networking/topic-8-kubernetes-networking.md). В production выбор CNI влияет на NetworkPolicy, метрики и отладку сети.

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
- [Networking: 8. Kubernetes Networking](../networking/topic-8-kubernetes-networking.md)
- [Traefik Ingress](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/)
