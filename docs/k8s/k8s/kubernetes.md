# Kubernetes
![Kubernetes[200]](./images/welcome.webp)

**Kubernetes (K8s)** — оркестратор контейнеров: автоматизирует развёртывание, масштабирование и управление приложениями в контейнерах. Кластер состоит из **Control Plane** (API, etcd, scheduler, контроллеры) и **нод** (kubelet, kube-proxy, container runtime); приложения описываются объектами API (Pod, Service, Deployment и т.д.) и поддерживаются контроллерами в желаемом состоянии.
## Разделы

- [**1. База: Linux, сеть и контейнеры**](topic-1-basis.md) — почему Kubernetes работает именно так: namespaces и cgroups, iptables/nftables, TCP/IP, DNS, NAT, балансировка L4/L7, TLS/mTLS; образы (слои, ENTRYPOINT/CMD, multi-stage, безопасность), OCI runtime (containerd, runc). Закрытие пробелов перед углублением в K8s.

- [**2. Архитектура Kubernetes**](topic-2-architecture.md) — Control Plane (kube-apiserver, etcd, quorum, backup/restore, scheduler, controller-manager, cloud-controller-manager); компоненты узла (kubelet, kube-proxy, CNI, CSI, CRI/containerd); практика: развернуть кластер (kind, k3s, kubeadm).

- [**3. Workloads и API-объекты**](topic-3-workloads-api.md) — Pod (lifecycle, probes, init/sidecar, ephemeral); ConfigMap, Secret; Service (ClusterIP, NodePort, LoadBalancer, Headless, ExternalName); PV/PVC; Resource Quotas, Limit Ranges, QoS, scheduler; RBAC; контроллеры (ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob).

- [**4. Сеть в Kubernetes**](topic-4-networking.md) — Pod-to-Pod, Service и kube-proxy (iptables/ipvs), DNS (CoreDNS), Ingress и TLS (Cert-Manager), Ingress Controllers (NGINX, Traefik), Gateway API, CNI (Calico, Cilium, Flannel); практика: Ingress + TLS, проверка из пода.

- [**5. Безопасность**](topic-5-security.md) — RBAC (Role, ClusterRole, RoleBinding, ServiceAccount), Pod Security (PSS, SecurityContext), NetworkPolicies и Zero Trust, Secrets и External Secrets/Vault; практика: ограничение доступа к namespace, NetworkPolicy deny all.

- [**6. Хранилище**](topic-6-storage.md) — PV и PVC, связывание, статическое и динамическое выделение, StorageClass (AccessModes, ReclaimPolicy), CSI и снимки, StatefulSet + PVC, best practices и production-пример (PostgreSQL).

- [**7. Наблюдаемость (Observability)**](topic-7-observability.md) — мониторинг (Prometheus, kube-state-metrics, Node Exporter), алертинг (Alertmanager, SLO/SLA/SLI), логирование (EFK/ECK/Loki, stdout), трейсинг (OpenTelemetry, Jaeger/Tempo); практика: алерты PodCrashLooping и NodeNotReady.

- [**8. Автомасштабирование и отказоустойчивость**](topic-8-autoscaling-resilience.md) — HPA, VPA, Cluster Autoscaler, KEDA, probes (liveness/readiness/startup), PodDisruptionBudget, Affinity/Anti-affinity, Taints и Tolerations.

- [**9. CI/CD и GitOps**](topic-9-cicd-gitops.md) — Helm (структура чарта, values, шаблоны, hooks), GitOps (Argo CD, Flux, стратегии синхронизации), CI (сборка образа, сканирование, деплой); практика: Git → CI → Helm → Argo CD → Cluster.

- [**10. Отладка и эксплуатация в production**](topic-10-troubleshooting-production.md) — типичные проблемы (Pending, ImagePullBackOff, OOMKilled, NetworkPolicy, etcd), инструменты (kubectl debug, stern, k9s, kubectl top, tcpdump), сценарии (падение ноды, недоступность API, массовый рестарт подов).

- [**11. Архитектура и Best Practices**](topic-11-architecture-best-practices.md) — multi-namespace, multi-cluster, Blue/Green и Canary, оптимизация затрат, обновление кластера, Disaster Recovery.

## Другое

- [Helm](../helm/helm.md) — пакетный менеджер для Kubernetes.
