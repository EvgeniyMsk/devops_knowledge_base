# 3. Архитектура Ansible‑проекта: roles, структура репозитория и переиспользуемость

---
Как строить Ansible‑проекты (Middle → Middle+): правильно организовывать роли, структуру репозитория, переиспользование через Ansible Galaxy и коллекции, разделение `roles/`, `group_vars/`, `inventories/`. В разделе — небольшие примеры кода, комментарии и production best practices.

---

## Roles: структура и best practices

**Role** — переиспользуемый модуль конфигурации: задачи, шаблоны, файлы, handlers и переменные, собранные в стандартной структуре.

### Инициализация роли (создание каркаса)

Самый простой и «правильный» способ начать новую роль — сгенерировать её структуру через **Ansible Galaxy**. Это сразу создаёт стандартные папки (`tasks/`, `defaults/`, `handlers/`, `meta/`, `templates/`, `files/`) и минимальные файлы.

```bash
# Создать роль в каталоге roles/
cd ansible
ansible-galaxy role init roles/nginx

# Или создать роль в текущей директории (по умолчанию создаст папку с именем роли)
ansible-galaxy role init nginx
```

После инициализации обычно:

- заполняют `defaults/main.yml` (конфигурируемые дефолты)
- добавляют `tasks/main.yml` (идемпотентные задачи)
- добавляют `handlers/main.yml` (restart/reload через notify)
- прописывают зависимости в `meta/main.yml` (если роль требует, например, `common`)
- при необходимости добавляют `templates/*.j2` и `files/*`

Production best practices:

- держать роль «как продукт»: `README.md` (что делает, vars, примеры), версии и changelog (если роль переиспользуется между проектами)
- не начинать роль с «копипасты» из playbook — лучше сначала создать каркас (`ansible-galaxy role init`), затем переносить логику небольшими шагами

### Базовая структура роли

```text
roles/
  nginx/
    defaults/main.yml   # дефолтные переменные (низкий приоритет)
    vars/main.yml       # переменные роли (высокий приоритет; использовать осторожно)
    tasks/main.yml      # входная точка задач
    handlers/main.yml   # handlers (restart/reload)
    templates/          # *.j2
    files/              # статические файлы
    meta/main.yml       # зависимости роли и метаданные (Galaxy)
```

### defaults vs vars (важное правило)

- **defaults/** — «безопасные дефолты», которые удобно переопределять на уровне окружений (`group_vars`, `host_vars`) или в playbook.
- **vars/** — жёсткие переменные роли с высоким приоритетом. В production чаще стараются **не** класть туда то, что нужно конфигурировать, чтобы не блокировать переопределения.

```yaml
# roles/nginx/defaults/main.yml
nginx_listen_port: 80
nginx_server_name: "_"
```

### Пример: роль nginx (tasks + template + handler)

```yaml
# roles/nginx/tasks/main.yml
- name: Install nginx
  become: true
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true

- name: Deploy nginx site config
  become: true
  ansible.builtin.template:
    src: site.conf.j2
    dest: /etc/nginx/sites-available/site.conf
    mode: "0644"
  notify: Reload nginx

- name: Enable site
  become: true
  ansible.builtin.file:
    src: /etc/nginx/sites-available/site.conf
    dest: /etc/nginx/sites-enabled/site.conf
    state: link
  notify: Reload nginx
```

```jinja2
{# roles/nginx/templates/site.conf.j2 #}
server {
  listen {{ nginx_listen_port }};
  server_name {{ nginx_server_name }};
  location / { return 200 "ok\n"; }
}
```

```yaml
# roles/nginx/handlers/main.yml
- name: Reload nginx
  become: true
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

Production best practices:

- роль должна быть **idempotent** и не делать лишних restarts (через handlers)
- роли должны быть **маленькими и сфокусированными** (KISS): одна роль — один компонент
- использовать `meta/main.yml` для зависимостей (например, `common`), а не копировать код

---

## Структура репозитория: roles/ + inventories/ + group_vars/

Рекомендуемая «проектная» структура:

```text
ansible/
  ansible.cfg
  site.yml
  roles/
    common/
    nginx/
    app/
    monitoring/
  inventories/
    dev/
      hosts.ini
      group_vars/
    staging/
      hosts.ini
      group_vars/
    prod/
      hosts.ini
      group_vars/
  group_vars/           # опционально: общие переменные (не зависят от окружения)
  host_vars/            # опционально
```

**Идея:** окружения отличаются inventory и vars, а роли остаются общими и переиспользуемыми.

### Пример site.yml, собирающий роли

```yaml
- name: Common baseline
  hosts: all
  become: true
  roles:
    - common

- name: Web layer
  hosts: web
  become: true
  roles:
    - nginx
    - app

- name: Monitoring
  hosts: monitoring
  become: true
  roles:
    - monitoring
```

Production best practices:

- `common` — базовая настройка ОС (пакеты, time sync, users/ssh hardening, логирование)
- роли `app`/`nginx`/`monitoring` держать независимыми, чтобы их можно было применять отдельно
- окружение (dev/stage/prod) — разные vars и лимиты, но **одни и те же роли**

---

## Переиспользуемость: Ansible Galaxy и коллекции

**Ansible Galaxy** помогает:

- использовать готовые роли/коллекции (`ansible-galaxy install ...`)
- публиковать свои роли (внутренняя Galaxy/репозиторий)

### Пример: requirements.yml

```yaml
roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
collections:
  - name: community.general
    version: ">=8.0.0"
```

```bash
ansible-galaxy role install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

Production best practices:

- **фиксировать версии** ролей/коллекций (иначе неожиданные изменения при обновлении)
- проверять качество внешней роли (идемпотентность, поддержка ОС, безопасность)
- для критичных ролей — форк/зеркало во внутреннем репозитории

---

## Практика: разбить большой playbook на роли

Цель практики: превратить «монолитный» playbook в проект с ролями:

- `common` — базовые пакеты, пользователи, ssh
- `nginx` — установка и конфиг Nginx
- `app` — деплой приложения (конфиг, systemd unit, healthcheck)
- `monitoring` — node_exporter, конфиг, сервис

Мини‑пример: роль `common` (users + ssh ключи) и её вызов.

```yaml
# roles/common/tasks/main.yml
- name: Ensure ops user exists
  ansible.builtin.user:
    name: ops
    groups: sudo
    append: true
    create_home: true

- name: Add authorized key for ops
  ansible.builtin.authorized_key:
    user: ops
    key: "{{ lookup('file', 'files/ops_id_rsa.pub') }}"
```

---

## Best practices для production (сводка)

- **Структура важнее «магии»**: роли + окружения в inventories
- **Минимум дублирования**: переиспользование через роли и зависимости
- **Версии зависимостей фиксировать**: requirements.yml + ревью обновлений
- **Роли писать как продукт**: defaults, handlers, корректный changed/failed, документация (README роли)

