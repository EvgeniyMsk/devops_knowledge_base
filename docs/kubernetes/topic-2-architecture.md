# 2. Архитектура Kubernetes

**Цель:** понимать внутреннее устройство кластера и уметь объяснять его на собеседовании: компоненты Control Plane (API, etcd, scheduler, controller-manager), компоненты узлов (kubelet, kube-proxy, CNI, CSI, CRI). Уметь развернуть кластер локально (kind, k3s) и хотя бы раз — через kubeadm.

---

## Control Plane

Компоненты управляющей плоскости отвечают за хранение состояния, принятие решений о размещении подов и поддержание желаемого состояния (контроллеры). Обычно работают на выделенных нодах (master/control-plane) или в подах (self-hosted / managed).

### kube-apiserver

**kube-apiserver** — единственная точка входа в Control Plane для всех запросов (kubectl, kubelet, контроллеры, веб-хуки). Принимает REST-запросы, валидирует их, читает и пишет состояние в etcd, координирует аутентификацию и авторизацию (RBAC). Все остальные компоненты plane с etcd не общаются напрямую — только через API. Высокая доступность достигается несколькими репликами apiserver за балансировщиком.

### etcd

**etcd** — распределённое key-value хранилище, в котором лежит всё состояние кластера (объекты API: Pod, Service, ConfigMap и т.д.). Kubernetes использует etcd как единственный источник истины. Данные хранятся в формате ключ-значение; версионность позволяет watch и согласованное чтение.

- **Quorum** — для корректной работы нужен кворум (большинство узлов). При N узлах etcd допустимо отказ (N/2) округлённо вниз узлов. Типично 3 или 5 узлов: при одном или двух падениях кластер etcd остаётся работоспособным.
- **Backup / restore** — состояние кластера критично; регулярный бэкап etcd и проверка восстановления обязательны для production. Инструменты: etcdctl snapshot save/restore, в managed-кластерах — сервис бэкапов провайдера.

### kube-scheduler

**kube-scheduler** — выбирает **узел**, на котором будет запущен под. Смотрит на запросы пода (resources, nodeSelector, affinity, taints/tolerations), на состояние узлов (ресурсы, условия) и назначает под на одну ноду. Саму запись в etcd и запуск контейнеров не делает — только выставляет поле `nodeName` в Pod; дальше за под отвечает kubelet на выбранной ноде.

### kube-controller-manager

**kube-controller-manager** — набор контроллеров в одном процессе. Каждый контроллер следит за определённым типом ресурсов и «подтягивает» текущее состояние к желаемому (reconciliation). Примеры: ReplicationController/ReplicaSet (держать N реплик пода), Node Controller (реакция на недоступные ноды), Endpoints Controller (заполняет Endpoints для Service), Namespace lifecycle и др. Контроллеры читают объекты через API и обновляют их (или создают связанные объекты), не запуская контейнеры напрямую — это делает kubelet.

### cloud-controller-manager

**cloud-controller-manager** — контроллеры, завязанные на API облачного провайдера: загрузка узлов (Node), балансировщики (LoadBalancer Service), маршруты и т.д. В bare metal или локальных кластерах (kind, k3s, kubeadm без cloud provider) этот компонент может не использоваться или быть пустым. В облаке (AWS, GCP, Azure) он нужен, чтобы Service типа LoadBalancer создавал реальный балансировщик в облаке.

---

## Компоненты узла (Node Components)

На каждой рабочей ноде работают агенты, которые запускают поды, настраивают сеть и хранилище.

### kubelet

**kubelet** — агент на узле. Получает список подов, назначенных на эту ноду (через apiserver или через локальный механизм, например static Pods), и обеспечивает запуск контейнеров через **CRI** (Container Runtime Interface). Следит за здоровьем контейнеров (liveness/readiness probe), монтирует тома (через CSI и другие интерфейсы), отчитывается о состоянии пода и ноды в API. Без kubelet нода «мёртвая» для кластера — поды на ней не запустятся.

### kube-proxy

**kube-proxy** — на каждой ноде поддерживает правила сетевой маршрутизации для Service (ClusterIP, NodePort). Реализует виртуальный IP сервиса: при обращении к ClusterIP:port трафик перенаправляется на один из эндпоинтов (под). Режимы работы: iptables, ipvs. Не занимается L7-маршрутизацией — только L4 (IP и порт).

### CNI plugin

**CNI** (Container Network Interface) — плагин, который настраивает сеть для подов: выделяет IP из пула подсети подов, создаёт veth/мосты или overlay, прописывает маршруты. Без CNI поды не получают IP и не могут общаться. Популярные плагины: Calico, Cilium, Flannel. Конфигурация обычно в /etc/cni/net.d/ и бинарники в /opt/cni/bin/.

### CSI plugin

**CSI** (Container Storage Interface) — стандарт для подключения томов к подам. Драйвер (CSI plugin) работает как набор подов в кластере и на нодах; по запросу PersistentVolume/PersistentVolumeClaim создаёт том в хранилище (облако, SAN, локальный диск) и монтирует его в под. Без CSI (или встроенного in-tree провайдера) можно использовать только emptyDir и часть hostPath; для stateful-нагрузок нужен CSI или аналог.

### CRI (containerd)

**CRI** (Container Runtime Interface) — интерфейс между kubelet и контейнерным runtime. Kubelet не вызывает Docker или containerd напрямую — он шлёт запросы по CRI (запустить под, остановить контейнер, логи и т.д.). **containerd** — типичный runtime в современных кластерах; он тянет образы из registry, создаёт контейнеры через **runc** (OCI runtime). Docker как runtime в K8s deprecated; образы остаются совместимыми (OCI), а запуском занимается containerd (или CRI-O).

---

## Практика: развернуть кластер

### kind (Kubernetes in Docker)

**kind** — кластер внутри Docker-контейнеров на одной машине. Удобен для локальной разработки и быстрых тестов.

```bash
# Установка (пример для Linux/macOS)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind && mv ./kind /usr/local/bin/

# Создать кластер
kind create cluster

# Контекст kubectl переключится на kind-kind
kubectl cluster-info
```

### k3s

**k3s** — облегчённый Kubernetes (один бинарник, меньше зависимостей). Подходит для edge, dev и лёгкого поднятия кластера на одной ноде.

```bash
# Установка
curl -sfL https://get.k3s.io | sh -

# kubeconfig для kubectl
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
```

### kubeadm (минимум один раз)

**kubeadm** — официальный инструмент для бутстрапа кластера (control plane + узлы). Даёт понимание, как поднимаются компоненты plane и как присоединяются ноды. Минимум один раз стоит пройти: init на первой ноде, join на остальных, установка CNI (например, Calico или Flannel).

```bash
# На control-plane ноде (после установки kubeadm, kubelet, kubectl)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Вывод команды содержит kubeadm join ... для рабочих нод
# Настройка kubeconfig для текущего пользователя
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Установка CNI (пример — Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

На рабочих нодах: установить kubeadm, kubelet, kubectl и выполнить `kubeadm join <args>` из вывода init. После этого `kubectl get nodes` покажет все ноды.

!!! tip "Практика"

    Для собеседования достаточно чётко назвать компоненты plane и узлов и их роли: API — вход и согласованность; etcd — состояние; scheduler — куда ставить под; controller-manager — поддержание желаемого состояния; kubelet — запуск подов на ноде; kube-proxy — Service; CNI — сеть подов; CRI — контейнеры. Плюс один раз поднять кластер через kubeadm или kind/k3s.

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Знать поток запроса** | kubectl → apiserver → etcd; scheduler выставляет nodeName; kubelet на ноде через CRI запускает контейнеры; CNI даёт поду IP. |
| **Бэкапить etcd** | В production обязательны регулярные снимки и проверка restore. |
| **Один раз собрать кластер вручную** | kubeadm даёт понимание компонентов и порядка запуска. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Путать kubelet и kube-proxy** | kubelet — поды и контейнеры; kube-proxy — только правила для Service. | Чётко разделять: kubelet = runtime на ноде, kube-proxy = сеть Service. |
| **Забывать про CNI после kubeadm init** | Без CNI поды в состоянии Pending (нет сети). | Всегда ставить CNI после init. |

---

## Дополнительные материалы

- [Kubernetes — Components](https://kubernetes.io/docs/concepts/overview/components/)
- [etcd — documentation](https://etcd.io/docs/)
- [kubeadm — create cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [kind](https://kind.sigs.k8s.io/)
- [k3s](https://k3s.io/)
- [CRI](https://kubernetes.io/docs/concepts/architecture/cri/)
- [CNI](https://www.cni.dev/)
- [CSI](https://kubernetes-csi.github.io/)
