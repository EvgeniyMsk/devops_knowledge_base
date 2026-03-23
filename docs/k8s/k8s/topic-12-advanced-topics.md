# 12. Дополнительные материалы

## Установка Gateway API

### Установка Envoy Gateway в кластер

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.1 \
  -n envoy-gateway-system \
  --create-namespace \
  --set kubernetesClusterDomain=k8s-cluster-dev.local
```

Команда устанавливает Envoy Gateway в Kubernetes через Helm из OCI-репозитория.

#### Что означает каждый параметр

- `helm install eg` — устанавливает релиз Helm с именем `eg`.
- `oci://docker.io/envoyproxy/gateway-helm` — источник чарта `gateway-helm` в Docker Hub (OCI registry).
- `--version v1.7.1` — фиксирует конкретную версию чарта.
- `-n envoy-gateway-system` — устанавливает релиз в namespace `envoy-gateway-system`.
- `--create-namespace` — создает namespace, если он еще не существует.
- `--set kubernetesClusterDomain=k8s-cluster-dev.local` — задает домен кластера.

#### Зачем указывать `kubernetesClusterDomain`

По умолчанию Kubernetes использует `cluster.local`.  
В этом кластере используется `k8s-cluster-dev.local`, поэтому параметр нужен для корректной работы внутренних DNS-имен сервисов, которые использует Envoy Gateway.

### Установка GatewayClass и Gateway

gateway.yaml

```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: yournamespace
spec:
  gatewayClassName: eg
  addresses:
    - type: IPAddress
      value: 98.131.121.45
  listeners:
    - name: http-tester
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https-tester
      protocol: HTTPS
      port: 443
      hostname: grafana.site.io
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: secret-tls-grafana
      allowedRoutes:
        namespaces:
          from: All
```

### Установка HTTPRoute

route.yaml

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend
  namespace: younamespace
spec:
  parentRefs:
    - name: eg
      namespace: yournamespace
      sectionName: https-tester
  hostnames:
    - "grafana.site.io"
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: dep-grafana
          port: 3000
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend-redirect-to-https
  namespace: yournamespace
spec:
  parentRefs:
    - name: eg
      namespace: yournamespace
      sectionName: http-tester
  hostnames:
    - "grafana.site.io"
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```

### Подключение basic auth

basic-auth.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-users
  namespace: yournamespace
type: Opaque
stringData:
  .htpasswd: |
    admin:{SHA}W6ph5Mm5Pz8GgiULbPgzG37mj9g=
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: tester-basic-auth
  namespace: yournamespace
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: backend
  basicAuth:
    users:
      name: basic-auth-users
```

