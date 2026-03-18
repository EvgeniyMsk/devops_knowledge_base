# 7. Работа с Helm и Kustomize

---
Эта тема учит применять **Helm** и **Kustomize** в Argo CD так, чтобы деплои были воспроизводимыми, безопасными и удобными для production: работа с `values`, override’ами, секретами в private репозиториях, overlay’ами и патчами.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Helm chart** | Пакет Kubernetes-манифестов с шаблонами и `values`. |
| **values override** | Переопределения параметров Helm для конкретного окружения (dev/stage/prod). |
| **Private repository** | Репозиторий, доступ к которому требует credentials (токен/ssh/username/password). |
| **Kustomize overlay** | Набор изменений поверх базового манифеста (реплики, ingress, config, параметры). |
| **patches** | Техника Kustomize для точечных изменений ресурсов. |
| **Argo CD source** | Блок `spec.source` в Application: repo/pathtype/helm/kustomize настройки. |

---

## Helm в Argo CD: values и воспроизводимость

### Production best practice: values только через окружения

- `values.yaml` в chart — значения по умолчанию
- окружения хранят свои `values-<env>.yaml`
- Argo CD подставляет конкретный values-file в зависимости от Application

Пример `Application` для Helm:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://git.example.com/platform/helm-charts.git
    targetRevision: v1.8.3
    chart: myapp
    helm:
      # production-подход: явный файл с параметрами окружения
      valueFiles:
        - environments/prod/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      # production чаще manual, но как пример оставим
      selfHeal: false
      prune: true
```

Комментарий:

- фиксируйте `targetRevision` для воспроизводимости;
- значения секретов в values лучше не хранить в open виде (см. Sealed Secrets/External Secrets).

---

### Helm: private репозитории (образы/чарты)

Если chart лежит в private repo (или зависимости/OCI require auth), Argo CD использует Kubernetes secrets/credentials.

Production best practice:

- credentials выдавайте только тем Application’ам/проектам, которым нужно;
- не храните token в plain-text в репозитории.

Идея (концептуально): вы добавляете credential в Argo CD (через Secret) и Argo CD использует его при pull.

---

## Kustomize в Argo CD: overlays и patches

### Production best practice: overlays для каждого окружения

Рекомендуемая структура:

```text
apps/
  myapp/
    base/
    overlays/
      dev/
      staging/
      prod/
```

Пример Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://git.example.com/platform/k8s-manifests.git
    targetRevision: main
    path: apps/myapp/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

---

### Пример Kustomize patch (концепт)

Идея: точечно меняем Deployment для окружения.

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base

patches:
  - path: replicas.patch.yaml
```

```yaml
# overlays/prod/replicas.patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

Best practice:

- делайте patches небольшими и предсказуемыми;
- избегайте “магии” в base, которую потом сложно объяснить в prod.

---

## Production чеклист для Helm/Kustomize в Argo CD

- Всегда документируйте, где “окружение” задаётся: `values` или overlay’ами.
- Проверяйте diffs перед применением (особенно когда меняете values/patches).
- Не храните секреты в values open-text.
- Фиксируйте `targetRevision` (или используйте релизные теги), чтобы деплой был воспроизводим.
- Делайте окружения раздельными: dev/staging/prod не должны делить одинаковые namespace/policy.

---

## Дополнительные материалы

- [Argo CD — Helm](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Argo CD — Kustomize](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/)
