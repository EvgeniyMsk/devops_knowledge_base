# 7. Провижининг и GitOps

---
Чтобы Grafana в **production** не превратилась в «музей правок в UI без истории», **дашборды** и **datasource’ы** выносят в **Git** и доставляют через **provisioning YAML**, **HTTP API** или **Terraform**. Ниже — структура каталогов, примеры конфигов, CI/CD и ограничения каждого подхода.

---

## Три уровня «как код»

| Подход | Плюсы | Минусы |
|--------|--------|--------|
| **Файловый provisioning** | Просто, прозрачно в Git | Часто нужен **restart** или watcher; алерты/папки — по версии Grafana |
| **HTTP API** | Гибко, скрипты миграций | Нужны **токены**, идемпотентность вручную |
| **Terraform** | Единый контур IaC, план/апплай | Дрейф, если кто-то правит UI в обход state |

**Best practice:** для командной Grafana правило «правки только через MR в Git» + **readonly** `editable: false` на критичных datasource’ах.

---

## Каталоги provisioning

Типичная раскладка в контейнере / на VM:

```text
/etc/grafana/provisioning/
├── dashboards/
│   └── dashboards.yaml      # куда смотреть за JSON дашбордов
├── datasources/
│   └── datasources.yaml
└── alerting/                  # при unified alerting (версии Grafana уточняйте)
    └── ...
```

В **Kubernetes** те же файлы монтируют **ConfigMap** или собирают в образ.

---

## Datasources (YAML)

```yaml
# /etc/grafana/provisioning/datasources/main.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: prom-prod
    type: prometheus
    access: proxy
    url: http://prometheus.monitoring.svc:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      timeInterval: 30s
    editable: false

  - name: Loki
    uid: loki-prod
    type: loki
    access: proxy
    url: http://loki.monitoring.svc:3100
    editable: false
```

**Комментарий:** фиксируйте **`uid`** — на него ссылаются дашборды при экспорте; смена UID ломает ссылки.

Секреты (**basic auth**, **token**) в **`secureJsonData`** и из **env** (`GF_SECURITY_…`) или vault-sidecar, не в открытом Git.

---

## Dashboards (YAML + JSON)

Провайдер указывает **папку** или **путь**, откуда подхватывать `*.json`:

```yaml
# /etc/grafana/provisioning/dashboards/from-git.yaml
apiVersion: 1

providers:
  - name: platform-dashboards
    orgId: 1
    folder: Platform
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: false
    options:
      path: /var/lib/grafana/dashboards/platform
```

- **`allowUiUpdates: false`** — после перезагрузки провижининг перезапишет бой (защита от правок только в UI без Git).
- JSON дашбордов кладут в **`/var/lib/grafana/dashboards/platform/*.json`** (volume / initContainer).

**Production:** в репозитории храните **один** дашборд — **один** файл; в MR виден **diff** панелей.

---

## HTTP API Grafana

Полезно для **разовых** миграций, синхронизации из скрипта или glue в пайплайне:

```bash
# Создать/обновить datasource (упрощённо; тело см. Swagger / docs)
export GF_URL="https://grafana.example.com"
export GF_TOKEN="glsa_..."  # Service token с правами admin или ограниченной роли API

curl -sS -X POST "${GF_URL}/api/datasources" \
  -H "Authorization: Bearer ${GF_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @datasource-prometheus.json
```

**Best practice:** **service account** + token с **минимальными** правами; не использовать пароль admin в CI.

Документация: [HTTP API](https://grafana.com/docs/grafana/latest/developers/http_api/).

---

## Terraform: провайдер Grafana

Один стиль с VPC, K8s и DNS:

```hcl
terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = "~> 3.0"
    }
  }
}

provider "grafana" {
  url  = var.grafana_url
  auth = var.grafana_token
}

resource "grafana_folder" "platform" {
  title = "Platform"
}

resource "grafana_dashboard" "nodes" {
  folder      = grafana_folder.platform.id
  config_json = file("${path.module}/dashboards/node-overview.json")
}
```

Ресурсы **datasource**, **alerting rule**, **contact point** зависят от версии провайдера — сверяйте с [registry](https://registry.terraform.io/providers/grafana/grafana/latest/docs).

**Production:** remote **state** с блокировкой; секреты из **TF_VAR** / CI variables; запрет «ручных» правок без импорта в state.

---

## Практика: дашборды в Git и CI/CD

### Принцип

1. **Экспорт** JSON из Grafana (или руками дежурного после согласованной правки) → коммит в **`dashboards/`**.
2. Пайплайн **валидирует** JSON (например `jq empty file.json`) и **деплоит**:
   - **вариант A:** обновить ConfigMap / артефакт в образе + rollout Grafana;
   - **вариант B:** `terraform apply` / вызов API.

### GitHub Actions (идея)

```yaml
# .github/workflows/grafana-dashboards.yml
name: sync-grafana-dashboards
on:
  push:
    branches: [main]
    paths: ["grafana/dashboards/**"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform apply
        env:
          TF_VAR_grafana_token: ${{ secrets.GRAFANA_TOKEN }}
        run: |
          cd terraform/grafana
          terraform init -backend-config=prod.hcl
          terraform apply -auto-approve
```

### GitLab CI (идея)

Тот же `terraform apply` в **job** с **protected** переменными `GRAFANA_TOKEN`, запуск только с `main`.

---

## Чеклист

| # | Практика |
|---|----------|
| 1 | **UID** datasource и дашбордов стабильны; описано в README репозитория |
| 2 | `editable: false` на datasource’ах, которые не должны «уехать» из UI |
| 3 | Ревью JSON дашбордов как кода (хотя бы дифф размера и ключевых запросов) |
| 4 | Один способ правды: Terraform **или** файлы + Helm, без дубля |
| 5 | Бэкап **БД Grafana** (если есть локальные правки) до массового `apply` |

---

## Дополнительные материалы

- [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Dashboards provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)
- [Terraform Grafana provider](https://registry.terraform.io/providers/grafana/grafana/latest/docs)
