# 5. Безопасность и доступы

---
Цель темы — научиться защищать Argo CD в production: ограничивать доступ к приложениям и репозиториям, интегрировать SSO (OIDC), а также правильно работать с секретами (Sealed Secrets, Vault/плагины). Отдельный акцент — least privilege и контроль доступа через AppProject и роли.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **RBAC** | Role-Based Access Control: политики “кто что может” (просматривать/синхронизировать/деплоить). |
| **AppProject** | Объект Argo CD, который ограничивает sourceRepos, destinations и (опционально) роли для приложений внутри периметра. |
| **Role / Policy** | Правила доступа (политики), которые решают, что разрешено конкретной роли/группе. |
| **OIDC** | OpenID Connect — стандарт SSO (например, Keycloak/Google/Auth0/Azure AD), используемый Argo CD для логина пользователей. |
| **Sealed Secrets** | Модель “зашифровать в репозитории, расшифровать в кластере”: controller расшифровывает `SealedSecret` в Kubernetes Secret. |
| **Vault** | Хранилище секретов; секрета выдаются через политику (token/policy) и обычно интегрируются через plugin или sidecar. |
| **Least privilege** | Минимально необходимые права: пользователю и сервису разрешаем ровно то, что нужно. |

---

## Production best practice: периметр через AppProject

Если у вас несколько команд/окружений, AppProject — это “граница”:

- какие репозитории разрешены (`sourceRepos`)
- в какие кластеры/namespace можно деплоить (`destinations`)
- кто и что может делать внутри периметра (roles/policies, при настройке)

Смысл: даже если у пользователя есть доступ к UI, он не должен иметь возможность деплоить “куда угодно”.

### Пример AppProject с ограничением репозиториев и destinations

```yaml
# AppProject ограничивает периметр для приложений.
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: "Platform apps"
  sourceRepos:
    - https://git.example.com/platform/*
  destinations:
    - server: https://kubernetes.default.svc
      namespace: production
    - server: https://kubernetes.default.svc
      namespace: staging
```

Best practice:

- делайте `sourceRepos` максимально узким (паттерны/regex — по возможности);
- destinations ограничивайте конкретными namespace, а не `*`.

---

## Ограничение доступа: роли dev / ops / admin

Argo CD использует политики (Casbin-подобная модель). Чаще всего:

- admin — полный доступ к управлению Argo CD и приложениям
- ops — deploy/refresh/rollback в рамках проектов/периметров
- dev — просмотр и self-service в рамках dev namespace (или отдельных приложений)

### Пример ролей внутри AppProject

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  sourceRepos:
    - https://git.example.com/platform/*
  destinations:
    - server: https://kubernetes.default.svc
      namespace: production
  roles:
    - name: dev
      description: "Dev can view apps in production read-only"
      groups:
        - dev-team
      policies:
        # Права “get” и “sync” задаются policy-шаблонами (зависит от вашей конфигурации).
        # Идея: dev не должен иметь права “deploy в production”.
        - p, proj:platform:dev, applications, get, platform/*, allow
    - name: ops
      description: "Ops can sync within platform"
      groups:
        - ops-team
      policies:
        - p, proj:platform:ops, applications, sync, platform/*, allow
```

Комментарий:

- точная форма policy зависит от того, как вы настраиваете RBAC/глобальные политики в Argo CD.
- best practice: начните с read-only для dev, затем расширяйте только после подтверждения процесса.

---

## Интеграция с SSO (OIDC)

Production best practice: не выдавайте локальные логины в Argo CD людям “из коробки”.

Типичный подход:

- настраиваем OIDC в Argo CD
- пользователи логинятся через корпоративный IdP
- роли назначаются по group claims (например, `dev-team`, `ops-team`)

### Мини-пример конфигурации OIDC (концептуально)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # В реальном проекте используйте секреты/переменные окружения, где необходимо.
  oidc.config: |
    name: ExampleSSO
    issuer: https://idp.example.com/
    clientID: argocd
    clientSecret: $OIDC_CLIENT_SECRET
    requestedScopes: ["openid","profile","email","groups"]
```

Best practice:

- `clientSecret` храните в Secret (или передавайте безопасно), не вplain-text в манифестах;
- сопоставляйте группы пользователей с Argo CD ролями (чёткий mapping).

---

## Secrets management: Sealed Secrets

Sealed Secrets удобен, когда:

- вы хотите хранить секреты в репозитории, но зашифрованными
- в кластере уже есть `sealed-secrets-controller`

### Пример SealedSecret

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
  namespace: production
spec:
  encryptedData:
    password: AgB...encrypted...
  template:
    metadata:
      name: db-password
      namespace: production
    type: Opaque
```

Best practice:

- разрешайте Argo CD применять SealedSecret, но не давайте прямой доступ на создание обычных Secrets из UI пользователям;
- следите за ротацией ключей Sealed Secrets (процесс зависит от контроллера).

---

## Secrets management: Vault (как обычно делают)

Vault в Argo CD чаще интегрируют одним из подходов:

- **plugin**: Argo CD рендерит манифесты/values, подставляя секреты на момент apply
- **внешняя синхронизация**: External Secrets/CSI driver заполняет Secrets в кластере, а Argo CD только применяет “безопасные” манифесты

Production best practice:

- не смешивайте “рендер секретов” и “логирование”: запретите утечки в stdout/stderr;
- используйте отдельные tokens/policies для команд, либо делайте секреты одинаковыми по scope, но строго разделёнными по окружениям.

---

## Практика: ограничить доступ к production deploy

Типовой “комбо”:

1. Включите protected branches и ограничьте права на production изменения.
2. В Argo CD:
   - разрешите sync только роли ops/admin для AppProject `platform`
   - dev оставьте read-only или вообще без destinations production

Мини-идея (workflow):

```text
dev: PR -> Git -> Argo CD показывает OutOfSync, но sync запрещён
ops: approval -> Argo CD sync в production
```

---

## Production чеклист по безопасности

- У Argo CD включён OIDC (SSO), а не “локальные логины для всех”.
- Роли (dev/ops/admin) ограничены через AppProject и policies.
- `sourceRepos` и `destinations` — максимально узкие.
- Секреты: Sealed Secrets или Vault/External Secrets; обычные Secrets не должны создаваться хаотично.
- Нет утечек секретов в логи job’ов (CI/CD) и в Argo CD.
- RBAC проверен на test namespace (попытки “deployer не имеет права” должны быть подтверждены практикой).

---

## Дополнительные материалы

- [Argo CD — RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)
- [Argo CD — OIDC](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#openid-connect-oidc)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Vault](https://developer.hashicorp.com/vault)

