# 9. Helm и CI/CD

---
На уровне **Middle+** chart **проверяется в пайплайне** (lint, рендер) и **деплоится** тем же способом, что согласован в команде: **GitLab CI**, **GitHub Actions** или оркестратор вокруг **`helm upgrade --install`**. Ниже — типовая схема стадий, минимальные YAML-примеры и production-замечания про доступ к кластеру.

---

## Что обычно автоматизируют

| Шаг | Зачем |
|-----|--------|
| **`helm dependency build` / `update`** | Собрать `charts/*.tgz` из `Chart.lock` в чистом агенте |
| **`helm lint`** | Структура chart’а и типовые ошибки шаблонов |
| **`helm template`** | Рендер с **prod-like** values без записи в кластер |
| **`helm upgrade --install`** | Деплой на stage/prod с **`--atomic`** / **`--wait`** |

Дополнительно (по политике): **`kubectl apply --dry-run=server`**, **kubeconform**, **policies** (OPA, Kyverno).

---

## GitLab CI: пример `.gitlab-ci.yml`

```yaml
stages:
  - verify
  - deploy

variables:
  HELM_VERSION: "3.14.4"
  CHART_DIR: chart

lint-chart:
  stage: verify
  image: alpine/helm:$HELM_VERSION
  script:
    - helm version
    - cd "$CHART_DIR" && helm dependency build || true
    - helm lint . -f ../ci/values-ci.yaml

template-chart:
  stage: verify
  image: alpine/helm:$HELM_VERSION
  script:
    - cd "$CHART_DIR"
    - helm dependency build || true
    - helm template gitlab-ci . -f ../ci/values-ci.yaml --debug > /tmp/renderedManifest.yaml
    - wc -l /tmp/renderedManifest.yaml

deploy-staging:
  stage: deploy
  image: alpine/helm:$HELM_VERSION
  before_script:
    - apk add --no-cache curl kubectl
    # kubeconfig: из CI/CD variable (файл) или kube-login по документации провайдера
    - mkdir -p ~/.kube && echo "$KUBECONFIG_B64" | base64 -d > ~/.kube/config
  script:
    - cd "$CHART_DIR" && helm dependency build || true
    - |
      helm upgrade --install "$HELM_RELEASE" . -n "$K8S_NAMESPACE" \
        --create-namespace \
        -f ../ci/values-staging.yaml \
        --set image.tag="$CI_COMMIT_SHORT_SHA" \
        --atomic --wait --timeout 10m
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

Комментарий: **`helm dependency build || true`** уместен, если зависимости уже **завендорены** в `charts/`; иначе уберите `|| true` и обеспечьте `helm repo add` + `helm dependency update`.

Best practices:

- в **MR** гонять только **verify** (lint + template), деплой — по ветке / тэгу;
- **не** печатать `values` с секретами;
- версию Helm **пинить** как образ агента.

---

## GitHub Actions: пример workflow

```yaml
name: helm-chart

on:
  push:
    branches: [main]
  pull_request:

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
        with:
          version: v3.14.4
      - name: Lint
        run: helm lint ./chart -f ci/values-ci.yaml
      - name: Template
        run: helm template ci ./chart -f ci/values-ci.yaml --debug

  deploy:
    needs: verify
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
        with:
          version: v3.14.4
      # Пример: AWS EKS через OIDC — замените на свой способ kubeconfig
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: eu-central-1
      - uses: azure/setup-kubectl@v4
      - name: Deploy
        run: |
          aws eks update-kubeconfig --name "$EKS_CLUSTER"
          helm upgrade --install my-app ./chart -n apps --create-namespace \
            -f ci/values-prod.yaml \
            --set image.tag="${{ github.sha }}" \
            --atomic --wait --timeout 10m
```

Комментарий: для **GKE/AKS** шаг с kubeconfig другой; смысл тот же: **краткоживущие** учётные данные (**OIDC**), не вечный token в секрете.

---

## Практика: минимальный pipeline

1. **`helm lint ./chart`** с одним файлом **ci/values-ci.yaml** (без секретов).
2. **`helm template test ./chart -f ci/values-ci.yaml`** в артефакте или только проверка exit code.
3. **`helm upgrade --install`** на staging с **`--atomic`**, **`--set image.tag=$GIT_SHA`**.

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | Одинаковая **версия Helm** локально и в CI |
| 2 | Деплой только из **защищённой** ветки / с approval |
| 3 | Доступ к кластеру через **RBAC** сервисного аккаунта CI, минимальные роли |
| 4 | Секреты через **Sealed/SOPS/Vault**, не через plain variables |
| 5 | После деплоя — **smoke** (HTTP check) или наблюдаемость в дашборде |

---

## Дополнительные материалы

- [Charts](https://helm.sh/docs/topics/charts/)
- [GitLab CI](https://docs.gitlab.com/ee/ci/)
- [GitHub Actions](https://docs.github.com/en/actions)

