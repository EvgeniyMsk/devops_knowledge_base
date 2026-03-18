# 11. Интеграции (CI → Argo CD и уведомления)

---

Интеграции нужны, чтобы:

- ускорять применение изменений (без ожидания очередного Git polling),
- и получать уведомления о sync/status в Slack и по email.

В production обычно стремятся к “быстро увидеть и быстро понять причину”, не создавая шум (лишние алерты/дубликаты).

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Git webhook** | HTTP-уведомление от Git-провайдера (GitHub/GitLab/Bitbucket) в Argo CD, чтобы он быстрее обновил состояние репозитория. |
| **`/api/webhook`** | Endpoint Argo CD для обработки Git webhook, который инициирует refresh приложений. |
| **`argocd-secret`** | Secret в namespace `argocd`, где Argo CD хранит ключи для проверки webhook (например, `webhook.gitlab.secret`). |
| **Argo CD Notifications** | Компонент для отправки уведомлений на каналы (Slack/Email/и т.п.) по событиям Argo CD. |
| **`argocd-notifications-cm`** | ConfigMap с определениями `trigger.*`, `template.*`, `service.*` и `subscriptions`. |
| **Trigger** | Условие, когда отправлять уведомление (например, sync succeeded/failed). |
| **Template** | Шаблон сообщения (например, Slack attachments или email subject/body). |
| **Subscription** | Связка “получатели → триггеры → шаблоны”, опционально с selector’ом по label’ам приложения. |

---

## CI → Argo CD: Git webhooks для ускорения sync

Argo CD по умолчанию опрашивает репозитории каждые ~3 минуты, чтобы обнаружить изменения. Git webhook позволяет “пнуть” API server и инициировать refresh сразу после `push`.

### Шаги для GitLab (production-путь)

1) В GitLab настройте webhook:

- **URL**: `https://argocd.example.com/api/webhook`
- **Secret (optional, но recommended)**: произвольная строка
- **Content type**: обычно `application/json` (для GitHub может быть критично; для GitLab обычно тоже удобно держать JSON)

2) В Kubernetes задайте секрет в `argocd-secret` (в namespace `argocd`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
stringData:
  # Секрет для проверки webhook GitLab
  webhook.gitlab.secret: "YOUR_GITLAB_WEBHOOK_SECRET"
```

3) (Опционально, но полезно) ограничьте размер payload’ов, чтобы снизить риск DoS:

```yaml
# Обычно задают через ConfigMap argocd-cm
data:
  webhook.maxPayloadSizeMB: "10" # production: меньше, чтобы отсечь слишком большие события
```

### Production best practices

- не держите Argo CD вебхук endpoint публичным без необходимости; по возможности используйте VPN/Ingress с auth,
- включайте общий секрет для GitLab webhook (пункт `webhook.gitlab.secret`),
- следите за отказоустойчивостью webhook: при проблемах Argo CD всё равно будет обновлять репозитории через polling (fallback),
- не полагайтесь на webhook как на единственный механизм доставки изменений: GitOps остаётся Git-first.

---

## Уведомления: Slack и Email (Argo CD Notifications)

Argo CD Notifications использует `argocd-notifications-cm` для:

- объявления триггеров (`trigger.*`),
- описания шаблонов сообщений (`template.*`),
- конфигурации сервисов доставки (`service.*`),
- указания subscriptions (получатели и какой набор триггеров обслуживать).

Ниже пример для:

- уведомлений при sync succeeded,
- уведомлений при sync failed,
- доставки в Slack и email.

### Пример `argocd-notifications-cm` (ConfigMap)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Триггер: успешный sync (по фазе операции)
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-sync-succeeded]

  # Триггер: проваленный sync (ошибка/failed)
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]

  # Шаблон для Slack: success
  template.app-sync-succeeded: |
    message: |
      ✅ {{ .app.metadata.name }} synced successfully.
      Revision: {{ .app.status.sync.revision | trunc 7 }}
      Details: {{ .context.argocdUrl }}/applications/{{ .app.metadata.name }}
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name }} synced",
          "title_link": "{{ .context.argocdUrl }}/applications/{{ .app.metadata.name }}",
          "color": "#18be52",
          "fields": [
            {"title": "Sync status", "value": "{{ .app.status.sync.status }}", "short": true},
            {"title": "Health", "value": "{{ .app.status.health.status }}", "short": true}
          ]
        }]

  # Шаблон для Slack: failed
  template.app-sync-failed: |
    message: |
      ❌ {{ .app.metadata.name }} sync FAILED.
      Error: {{ .app.status.operationState.message }}
      Details: {{ .context.argocdUrl }}/applications/{{ .app.metadata.name }}
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name }} sync FAILED",
          "title_link": "{{ .context.argocdUrl }}/applications/{{ .app.metadata.name }}",
          "color": "#E96D76",
          "fields": [
            {"title": "Operation phase", "value": "{{ .app.status.operationState.phase }}", "short": true},
            {"title": "Reason", "value": "{{ .app.status.operationState.message }}", "short": false}
          ]
        }]

  # Сервис доставки Slack (нужен token Slack)
  # В реальном deployment значения прокидывают через argocd-notifications-secret.
  service.slack: |
    token: $slack-token
    # username/icon можно добавить по желанию

  # Сервис email через SMTP
  service.email: |
    host: smtp.example.com
    port: 587
    from: no-reply@example.com
    username: $email-username
    password: $email-password

  # Контекст (URL вашего Argo CD UI)
  context: |
    argocdUrl: https://cd.example.com

  # Subscriptions: кому слать и какие триггеры включены
  subscriptions: |
    - recipients:
        - slack: "#platform-deployments"  # имя канала/получателя (зависит от вашей настройки)
        - email: devops@example.com
      triggers:
        - on-sync-succeeded
        - on-sync-failed
      # Production: при необходимости можно ограничить scope через selector по label’ам приложения
      # selector: notifications=enabled
```

### Пример `argocd-notifications-secret` (Secret с секретами)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  slack-token: "YOUR_SLACK_BOT_TOKEN"
  email-username: "smtp-user"
  email-password: "smtp-password"
```

### Production best practices (чтобы не утонуть в шуме)

- добавляйте `oncePer` (если захотите) для событий, которые могут повторяться одинаковой revision,
- ограничивайте scope selector’ом (например, label’ом `notifications=enabled` для production-critical apps),
- различайте severity: success можно слать часто, failure лучше отправлять более узко и с контекстом ошибки,
- не логируйте/не коммитьте реальные токены (держите их только в Secret’ах).

---

## Практика: настроить уведомления о sync/status

1) Разверните/убедитесь, что установлен Argo CD Notifications controller.
2) Примените `argocd-notifications-cm` и `argocd-notifications-secret`.
3) Добавьте label’ (если используете selector) в приложение Kubernetes:

```yaml
metadata:
  labels:
    notifications: enabled
```

4) Сделайте “контролируемый” тест:

- измените манифест в Git (чтобы триггернул sync),
- дождитесь результата,
- проверьте, что Slack/email получили сообщение,
- при failure сверяйте `operationState.message` в Argo CD.

Полезные команды для диагностики:

```bash
kubectl -n argocd get cm argocd-notifications-cm -o yaml
kubectl -n argocd logs deploy/argocd-notifications-controller --tail=100
```

---

## Production checklist

| Что сделать | Зачем |
|--------------|-------|
| Git webhook включён и защищён секретом | Быстрее обновления без ожидания polling. |
| `argocd-notifications-cm` содержит только необходимые триггеры | Меньше шума и дублирования. |
| Есть failure-путь с понятным error/context | Ускоряет MTTR. |
| Уведомления scoping’нуты selector’ом (при необходимости) | Защита от спама по некритичным сервисам. |

