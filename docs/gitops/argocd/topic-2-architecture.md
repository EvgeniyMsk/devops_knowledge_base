# 2. Архитектура ArgoCD

---
Эта тема помогает понять, как Argo CD работает внутри: какие компоненты отвечают за API, за чтение репозитория и за контроллер синхронизации, а также как именно выполняется `sync` в Kubernetes.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Argo CD** | GitOps-контроллер для Kubernetes: держит кластер в согласованности с Git-репозиторием. |
| **API Server** | Компонент Argo CD, который принимает запросы (UI/CLI) и управляет доступом к данным. |
| **Repo Server** | Компонент, который читает Git-репозиторий, собирает манифесты (helm/kustomize) и отдаёт результат контроллеру. |
| **Application Controller** | Контроллер, который сравнивает desired state (из Git) и actual state (в кластере) и запускает синхронизацию. |
| **Application** | CRD Argo CD: описание того, какой Git-репозиторий/путь деплоить и в какой cluster/namespace. |
| **Sync (синхронизация)** | Приведение фактического состояния к желаемому: применение манифестов из репозитория. |
| **Desired state** | То, что описано в Git (rendered manifests). |
| **Actual state** | То, что реально существует в Kubernetes. |
| **Diff (различия)** | Разница между desired и actual state, которая определяет, нужен ли sync. |
| **Health/Synchronization status** | Статусы приложения: “здоров” ли кластер, и “в синхронизации” ли он с Git. |

---

## Как компоненты Argo CD связаны между собой

Упрощённая схема:

```text
UI/CLI -> Argo CD API Server -> Application Controller
                              |
                              +-> Repo Server (Git fetch + render)
```

Production-вывод:

- если “не синхронизируется”, часто проблема в связке “Repo Server ↔ render” или в правах контроллера на Kubernetes API;
- если “не показываются изменения”, проверьте refresh/cache со стороны Repo Server.

---

## Что происходит на стороне Kubernetes

Argo CD работает как orchestrator:

1. Получает исходный манифест (rendered manifests) из Repo Server.
2. Делает diff с текущим состоянием в кластере.
3. Если включён sync — применяет изменения через Kubernetes API.
4. Следит за статусом ресурсов и обновляет health/sync status у Application.

Важно: Argo CD не “изменяет Git”, он изменяет **кластер**, чтобы тот соответствовал **Git**.

---

## Как работает sync: пошагово

1. **Application Controller** обновляет желаемое состояние: запрашивает данные у Repo Server (Git fetch, helm/kustomize render).
2. **Diff**: вычисляются различия между desired и actual.
3. **Pre-sync шаги** (если настроены): например, hooks (в зависимости от конфигурации приложения).
4. **Apply**: Argo CD применяет манифесты (по сути — через Kubernetes apply/patch логику, зависящую от режима).
5. **Post-sync и мониторинг**: контроллер ждёт готовности (health), фиксирует результат и обновляет UI/статусы.

---

## Практика: установка Argo CD

### Вариант 1: через Helm (рекомендуется для production)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argo-cd argo/argo-cd \
  --namespace argocd --create-namespace \
  --set server.service.type=ClusterIP
```

Best practice:

- в production держать `server.service.type=ClusterIP` и публиковать UI/API через Ingress (TLS, auth, WAF).
- ограничивать доступ к UI через NetworkPolicy и/или внешний auth-proxy.

---

### Вариант 2: через manifests (для обучения/простых стендов)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Комментарий:

- manifests-установка хороша для знакомства, но в production лучше GitOps/Helm-подход с фиксированными версиями.

---

## UI и CLI: что использовать

### UI (быстрое понимание состояния)

- смотрите `Sync Status` и `Health Status`,
- открывайте “Diff” для конкретного ресурса.

### CLI (управление и расследование)

Примеры команд:

```bash
# логин (после установки нужно получить admin password, зависит от схемы)
argocd login <ARGOCD_SERVER> --username admin --insecure

# получить список Application
argocd app list

# синхронизировать приложение
argocd app sync myapp

# дождаться результата (и увидеть, что именно пошло не так)
argocd app wait myapp --health
```

Best practice:

- все важные изменения делайте через Git (commit/PR).
- CLI используйте для запуска sync/диагностики, но не как “постоянный способ деплоя”.

---

## Production best practices по архитектуре

| Практика | Почему важно |
|----------|--------------|
| Обновляйте Argo CD с фиксированными версиями | Уменьшает риск несовместимостей и “сломанных” hooks/render. |
| Ограничивайте доступ к API/UI | UI — зона риска (секреты/управление). |
| Проверяйте права контроллера в кластере | Если controller не может применять — sync будет “успешным в diff, но неуспешным в apply”. |
| Следите за Repo Server | Render heavy workloads могут замедлять обновления. |
| Используйте отдельные namespaces и RBAC | Снижает blast radius и упрощает безопасность. |

---

## Дополнительные материалы

- [Argo CD — Concepts](https://argo-cd.readthedocs.io/en/stable/)
- [Argo CD — Application](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
