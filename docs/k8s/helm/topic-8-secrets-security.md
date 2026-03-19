# 8. Секреты и безопасность

---
В **values.yaml** и Git **нельзя** хранить пароли и ключи в открытом виде: любой с доступом к репозиторию или **`helm get values`** увидит секреты. В production обычно выбирают **Sealed Secrets**, **SOPS** (часто через плагин **helm-secrets**), **External Secrets** с **Vault** или другим бэкендом. Ниже — сравнение подходов и минимальные примеры «безопасного» деплоя.

---

## Plain `Secret` и открытые values — почему это плохо

| Риск | Деталь |
|------|--------|
| **Git** | История коммитов «помнит» секрет даже после удаления |
| **`helm get values`** | Values попадают в **Secret** релиза в namespace — см. RBAC |
| **Логи CI** | Случайный `echo` или артефакт с сырым YAML |

Допустимо только для **локальной** отладки и **одноразовых** учебных кластеров — и то без реальных учётных данных.

---

## Sealed Secrets

Идея: в Git лежит **`SealedSecret`**, зашифрованный **публичным** ключом контроллера в кластере. Расшифровать может только **тот** кластер (или согласованная ротация ключей).

### Учебный поток

```bash
# Сервер с установленным sealed-secrets-controller и kubeseal локально

kubectl create secret generic my-app \
  --from-literal=api-key='REDACTED' \
  --dry-run=client -o yaml \
  | kubeseal -o yaml > templates/sealedsecret-my-app.yaml   # путь внутри вашего chart
```

Далее `SealedSecret` можно **рендерить через Helm** как обычный шаблон (или хранить статично в `templates/`) и ставить **`helm upgrade`** — контроллер создаст обычный **`Secret`**.

Best practices:

- **ротация** ключей и runbook на аварийный сценарий;
- ограничить **RBAC** на `SealedSecret` и итоговый `Secret`;
- не коммитить **несшифрованный** `Secret`.

Документация: [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets).

---

## SOPS и плагин `helm-secrets`

**Mozilla SOPS** шифрует YAML/JSON (часто через **KMS** или **age**). Плагин **`helm-secrets`** позволяет передавать Helm **расшифрованные** values только на время выполнения на доверенной машине/агенте CI.

```bash
# Установка плагина (окружение команды может отличаться)
helm plugin install https://github.com/jkroepke/helm-secrets

# Пример вызова (файл values.enc.yaml зашифрован SOPS)
helm secrets upgrade --install my-app ./my-app -n apps \
  -f values.yaml \
  -f secrets.enc.yaml
```

Production:

- ключи **KMS/age** только через **OIDC** агента CI, не в переменной репозитория;
- **не** кэшировать расшифрованный файл в артефактах;
- проверить совместимость версии плагина с **Helm 3**.

---

## HashiCorp Vault и операторы

Подходы без хранения значения в Git вообще:

| Паттерн | Суть |
|---------|------|
| **Vault Agent Injector** | Токен/Sidecar подставляет секрет в Pod |
| **Vault CSI Provider** | Том с секретами, синхронизация |
| **External Secrets Operator** + Vault | CR `ExternalSecret` → обычный `Secret` в namespace |

Helm при этом ставит **приложение** и, при необходимости, **ServiceAccount** / политику доступа; **содержимое** секрета живёт в Vault.

Комментарий: см. материалы по Vault в разделе **«Хранилище секретов»** этого сайта — та же модель доверия и **least privilege**.

---

## Практика: задеплоить секрет безопасно (выберите один контур)

### Вариант A — Sealed Secret

1. Установить **Sealed Secrets controller** в кластер (отдельный Helm chart / манифесты платформы).
2. Получить `kubeseal` и создать **`SealedSecret`** из `kubectl create secret ... --dry-run=client`.
3. Положить YAML в `templates/` или включить в chart как шаблон с **`{{ .Release.Namespace }}`** при необходимости.
4. `helm upgrade --install` — убедиться, что обычный `Secret` появился: `kubectl get secret -n apps`.

### Вариант B — SOPS + helm-secrets

1. Создать `secrets.yaml`, зашифровать **`sops -e -i secrets.yaml`** (или через `.sops.yaml` с KMS).
2. В CI: запуск **`helm secrets upgrade`** с ключом расшифровки.
3. В Git только **`secrets.enc.yaml`**.

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | Секреты **не** в открытом `values.yaml` и не в логах CI |
| 2 | **RBAC**: кто может читать `Secret` и `helm get values` |
| 3 | Включено **шифрование etcd** у провайдера (где применимо) |
| 4 | Ротация секретов и **отзыв** при утечке |
| 5 | Единый **стандарт** команды: Sealed / SOPS / Vault, а не «кто как успел» |

---

## Дополнительные материалы

- [Helm — chart best practices](https://helm.sh/docs/chart_best_practices/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [helm-secrets](https://github.com/jkroepke/helm-secrets)
- [Mozilla SOPS](https://github.com/getsops/sops)
- [External Secrets Operator](https://external-secrets.io/)

