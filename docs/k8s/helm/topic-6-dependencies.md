# 6. Зависимости чартов

---
Сложное приложение часто собирают из **нескольких chart'ов**: ваш «umbrella» chart подключает **Redis**, **PostgreSQL**, **ingress-nginx** и т.д. Зависимости описываются в **`Chart.yaml`**, подтягиваются командой **`helm dependency update`**, версии фиксируются в **`Chart.lock`**. Ниже — формат зависимостей, работа с репозиториями, передача **values** во вложенные chart’ы и production-практики.

---

## Блок `dependencies` в `Chart.yaml`

```yaml
apiVersion: v2
name: my-platform
description: Платформенный chart с зависимостями
type: application
version: 1.0.0
appVersion: "1.0"

dependencies:
  - name: redis
    version: "~18.19.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
  - name: nginx
    version: "15.14.0"
    repository: "https://kubernetes.github.io/ingress-nginx"
    alias: ingress-nginx
```

| Поле | Назначение |
|------|------------|
| **name** | Имя chart’а в репозитории |
| **version** | Диапазон или точная версия (semver); в production лучше **жёстче** |
| **repository** | URL **index.yaml** репозитория или **OCI** (`oci://...`) |
| **condition** | Включать subchart только если `values.yaml` задаёт `redis.enabled: true` |
| **alias** | Установить subchart под другим имени релиза в дереве values |

После изменения зависимостей:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm dependency update ./my-platform
```

В каталоге chart’а появится **`charts/`** с `.tgz` и файл **`Chart.lock`** — его **коммитят** в Git для воспроизводимых сборок.

Альтернатива без сети в CI:

```bash
helm dependency build ./my-platform
```

(использует **`Chart.lock`**, если артефакты уже доступны).

---

## Передача values во вложенный chart

Ключ верхнего уровня в **`values.yaml`** родителя совпадает с **именем subchart** (или с **`alias`**):

```yaml
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    existingSecret: redis-credentials
    existingSecretPasswordKey: password

ingress-nginx:
  controller:
    service:
      type: LoadBalancer
```

Комментарий: точная структура вложена в документацию **каждого** community chart’а — копируйте из их `values.yaml`, а не угадывайте.

---

## Практика: подключить Redis и Ingress NGINX

### 1. Подготовить `Chart.yaml` (фрагмент)

Зафиксируйте версии под свой регистр и политику безопасности (ниже — ориентир):

```yaml
dependencies:
  - name: redis
    version: 18.19.1
    repository: https://charts.bitnami.com/bitnami
  - name: ingress-nginx
    version: 4.11.2
    repository: https://kubernetes.github.io/ingress-nginx
    alias: ingress-nginx
```

Версии проверьте на **Artifact Hub** / в репозитории перед использованием.

### 2. Обновить зависимости

```bash
helm dependency update ./my-platform
helm dependency list ./my-platform
```

### 3. Рендер или установка

```bash
helm template my-release ./my-platform \
  -f values.yaml \
  -f values-prod.yaml \
  --debug | less

helm upgrade --install my-platform ./my-platform -n ingress \
  -f values-prod.yaml \
  --atomic --timeout 15m
```

Часто **ingress-nginx** ставят **отдельным** релизом один раз на кластер; подключение через dependency уместно в **учебном** umbrella или если политика платформы это допускает.

---

## Production best practices

| Практика | Зачем |
|----------|--------|
| **Chart.lock** в Git | Одинаковые версии subchart’ов в CI и prod |
| Пиновать **version** без широких `^` в prod | Предсказуемость и аудит |
| Сканировать **образы** и chart’ы (trivy, supply chain) | Community chart ≠ доверие по умолчанию |
| **condition** для опциональных БД | Не ставить Redis всем потребителям umbrella |
| Не дублировать то, что уже есть в кластере (**Redis Operator**, managed cache) | Меньше операционных затрат |

---

## Дополнительные материалы

- [Chart dependencies](https://helm.sh/docs/topics/charts/#chart-dependencies)
- [helm dependency](https://helm.sh/docs/helm/helm_dependency/)
- [Artifact Hub](https://artifacthub.io/) — поиск версий и репозиториев

