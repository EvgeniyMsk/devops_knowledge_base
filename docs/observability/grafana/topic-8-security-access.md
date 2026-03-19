# 8. Безопасность и доступы

---
Публичная Grafana с паролем **admin/admin** — прямой путь к утечке метрик, логов и внутренних топологий. В **production** нужны **роли**, **команды (Teams)**, **SSO** и **права на папки** и дашборды. Ниже — модель доступа Grafana, **OAuth** (GitHub, Google), ограничение дашбордов по командам и практика разделения **Dev / Ops / Viewer**.

---

## Пользователи, команды и роли

| Сущность | Назначение |
|----------|------------|
| **User** | Учётная запись (локальная или из SSO) |
| **Team** | Группа для выдачи прав на **папки**/датасорсы пакетно |
| **Role** (org-level) | **Viewer** — только просмотр; **Editor** — дашборды; **Admin** — орг., пользователи, datasource’ы |

**Best practice:** **Admin** в организации — минимальное число людей; для изменения дашбордов в проде часто достаточно **Editor** с **ограниченными папками**, а не глобального Editor.

**Service accounts** (токены) используйте для **API**, **Terraform**, **CI** — не для людей; ротация токенов и **least privilege**.

---

## OAuth: GitHub, Google (идея)

Единый вход снимает бремя локальных паролей и позволяет отключить **basic auth** в публичной сети.

### Фрагмент `grafana.ini` (GitHub)

```ini
[auth.github]
enabled = true
allow_sign_up = true
client_id = ${GF_AUTH_GITHUB_CLIENT_ID}
client_secret = ${GF_AUTH_GITHUB_CLIENT_SECRET}
scopes = user:email,read:org
auth_url = https://github.com/login/oauth/authorize
token_url = https://github.com/login/oauth/access_token
api_url = https://api.github.com/user
team_ids = <optional numeric team ids>
allowed_organizations = my-corp
```

### Фрагмент `grafana.ini` (Google)

```ini
[auth.google]
enabled = true
allow_sign_up = true
client_id = ${GF_AUTH_GOOGLE_CLIENT_ID}
client_secret = ${GF_AUTH_GOOGLE_CLIENT_SECRET}
scopes = openid email profile
auth_url = https://accounts.google.com/o/oauth2/v2/auth
token_url = https://oauth2.googleapis.com/token
allowed_domains = corp.example.com
```

**Production:**

- секреты только из **переменных окружения**/K8s Secret, не в Git;
- **`allow_sign_up`** при необходимости ограничить **`allowed_domains`** / **`allowed_organizations`**;
- включить **HTTPS** на Ingress; cookie **`secure`**.

Документация: [Configure GitHub OAuth](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/github/), [Google](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/google/).

---

## Права на папки (folder permissions)

- Каждый **дашборд** живёт в **папке** (или General).
- **Permissions** на папку: кто из **Users/Teams** — Viewer / Editor.
- Так изолируют **кластер A** vs **кластер B** или **команда платформы** vs **продуктовые** скрины.

**Сценарий:** команда **Payments** — **Editor** только в папке `Payments`; остальные папки — **Viewer** или **no access** (зависит от настроек «умолчания» и версии).

**Best practice:** не кладёте все дашборды в **General** без разбора — там сложнее разграничить и проще «забыть» публичный доступ.

---

## Практика: Dev / Ops / Viewer

| Роль в процессе | Роль в Grafana | Папки / замечание |
|-----------------|----------------|-------------------|
| **Viewer** (разработчик) | **Viewer** | Продуктовые и **staging** обзоры; без редактирования prod-дашбордов |
| **Dev** (владелец сервиса) | **Editor** в папке команды | MR на JSON в Git, если включён GitOps |
| **Ops / SRE** | **Editor** или **Admin** (орг.) | Платформенные дашборды, алерты, datasource **read** |

Для **staging** и **production** часто используют **две** инсталляции Grafana или **две организации** в одной — чтобы случайно не править prod из тестового UI.

---

## Данные и сеть

| Уровень | Практика |
|---------|----------|
| **Ingress** | Только TLS; по возможности **IP allowlist** или корпоративный VPN |
| **Datasource** | URL к Prometheus/Loki только во **внутренней** сети; credentials в **Secret** |
| **Аудит** | При требованиях compliance — включить **audit log** / прокси с логированием (зависит от редакции Grafana) |

---

## Чеклист

| # | Практика |
|---|----------|
| 1 | Отключены дефолтные слабые пароли; **admin** не для повседневной работы |
| 2 | **SSO** + ограничение домена/организации |
| 3 | **Teams** ↔ папки; ревью прав при онбординге |
| 4 | Service accounts для автomation с минимальными правами |
| 5 | Регулярный аудит: кто **Org Admin**, какие **datasource** editable |

---

## Дополнительные материалы

- [Role-based access control](https://grafana.com/docs/grafana/latest/administration/roles-and-permissions/)
- [Team sync](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-team-sync/)
- [Folder permissions](https://grafana.com/docs/grafana/latest/administration/user-management/manage-dashboard-permissions/)
