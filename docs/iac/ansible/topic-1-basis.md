# 1. База Ansible: inventory, playbooks, переменные и шаблоны

---
Понимать, как устроены inventory (static/dynamic), playbooks и flow выполнения, задачи и модули, переменные (group_vars/host_vars), шаблоны Jinja2, handlers и tags. В конце — практические мини‑примеры (Nginx + template, PostgreSQL с параметрами, пользователи и SSH) и best practices из production.

---

## Inventory (static / dynamic)

**Inventory** — список управляемых хостов и групп. Может быть:

- **static**: файл INI/YAML в репозитории
- **dynamic**: inventory-плагин/скрипт (например, AWS/GCP/Azure), который строит список хостов из API провайдера

### Пример: статический inventory (INI)

```ini
[web]
web-1 ansible_host=10.0.1.10 ansible_user=ubuntu
web-2 ansible_host=10.0.1.11 ansible_user=ubuntu

[db]
db-1 ansible_host=10.0.2.10 ansible_user=ubuntu

[prod:children]
web
db
```

### Пример: inventory (YAML)

```yaml
all:
  children:
    prod:
      children:
        web:
          hosts:
            web-1:
              ansible_host: 10.0.1.10
              ansible_user: ubuntu
            web-2:
              ansible_host: 10.0.1.11
              ansible_user: ubuntu
        db:
          hosts:
            db-1:
              ansible_host: 10.0.2.10
              ansible_user: ubuntu
```

### Production best practices

- **Держать inventory и переменные в Git**: воспроизводимость и ревью изменений.
- **Разделять окружения**: `inventories/dev`, `inventories/staging`, `inventories/prod`.
- **Не хранить секреты в открытом виде**: Ansible Vault или внешнее хранилище (Vault/Secrets Manager).
- **Dynamic inventory** использовать там, где инфраструктура меняется часто; статический — для стабильных серверов.

---

## Playbooks: структура и execution flow

Playbook — YAML‑файл с одним или несколькими **plays**. Каждый play выбирает `hosts`, задаёт `vars`, `roles` и `tasks`.

### Пример: минимальный playbook

```yaml
- name: Install and start nginx
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

### Как Ansible выполняет задачи

- По умолчанию задачи выполняются **последовательно внутри хоста**, но **параллельно по хостам** (параметр `forks`).
- `serial` позволяет катить изменения батчами (rolling update по нодам).
- `check mode` (`--check`) показывает, что бы изменилось, без применения (если модуль поддерживает).

```bash
# Прогон в check mode
ansible-playbook -i inventories/prod/hosts.ini site.yml --check

# Rolling по 20% хостов
ansible-playbook -i inventories/prod/hosts.ini site.yml --forks 20
```

### Production best practices

- **Idempotency**: playbook должен быть безопасен при повторном запуске.
- **serial + health checks** для критичных сервисов (обновлять по частям).
- **--diff** на этапе ревью/в CI (где уместно): видеть изменения шаблонов/файлов.

---

## Tasks и модули

Задачи (tasks) используют **модули**. Предпочтительно использовать модули (`apt`, `service`, `template`, `user`, `authorized_key`), а не `shell/command` — так проще обеспечить идемпотентность.

```yaml
- name: Create app user
  ansible.builtin.user:
    name: app
    shell: /bin/bash
    create_home: true
```

### Production best practices

- Минимизировать `shell`: если без него нельзя — добавлять `creates/removes` или проверки.
- Использовать `changed_when` / `failed_when` для корректной сигнализации результата.

---

## Переменные (group_vars, host_vars)

Переменные можно задавать на разных уровнях: в playbook, в inventory, в `group_vars/` и `host_vars/`.

### Способы задания переменных (самые частые)

- **В playbook**: `vars:` внутри play (удобно для локальных значений), либо `vars_files:` (подключение файлов с переменными).
- **В inventory**: переменные у хоста/группы прямо в inventory (INI/YAML) — удобно для небольших наборов, но в больших проектах хуже читаемость.
- **`group_vars/` и `host_vars/`**: основной способ для проектов — данные отделены от логики.
- **Extra vars** (`-e` / `--extra-vars`): самый высокий приоритет из обычных способов; удобно для CI (например, `release_version`), но не стоит превращать в «второй конфиг» проекта.
- **Facts** (`ansible_facts`): автоматически собранные данные о системе (OS, сеть и т.д.) или кастомные facts.
- **Set_fact**: вычисленные на лету значения (повышает «магичность», использовать осторожно).

Пример `vars_files` и `-e`:

```yaml
- name: Example vars
  hosts: web
  vars_files:
    - vars/common.yml
  vars:
    nginx_listen_port: 8080
  tasks:
    - ansible.builtin.debug:
        msg: "port={{ nginx_listen_port }}"
```

```bash
# Перезаписать значение через extra vars
ansible-playbook -i inventories/prod/hosts.ini site.yml -e nginx_listen_port=9090
```

### Пример структуры

```text
inventories/prod/hosts.ini
group_vars/web.yml
group_vars/db.yml
host_vars/db-1.yml
```

```yaml
# group_vars/web.yml
nginx_listen_port: 80
```

```yaml
# host_vars/db-1.yml
postgres_max_connections: 300
```

### Production best practices

- **Стандартизировать имена переменных** (префиксы по роли/сервису).
- **Разделять дефолты и секреты**: дефолты в `group_vars`, секреты — в Vault.

### Приоритеты переменных (упрощённо)

Полная схема приоритетов в Ansible большая, но для практики достаточно помнить порядок (от низкого к высокому):

1. **Role defaults** (`roles/<role>/defaults/main.yml`)
2. **Inventory vars** (group_vars/host_vars, vars в inventory)
3. **Play vars** (`vars:` в playbook, `vars_files:`)
4. **Task vars / set_fact** (локально по ходу выполнения)
5. **Extra vars** (`-e`) — почти всегда «побеждают»

Production best practices:

- конфигурируемые значения держать в **defaults** роли и переопределять через **group_vars/host_vars**
- **extra vars** использовать точечно (релизный тег, feature‑toggle), но не хранить там постоянную конфигурацию
- `set_fact` применять только когда действительно нужно вычисление на лету, и документировать, откуда берётся значение

---

## Templates (Jinja2)

Шаблоны позволяют генерировать конфиги из переменных.

### Пример: Nginx конфиг через template

```yaml
- name: Deploy nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: "0644"
  notify: Reload nginx
```

```jinja2
# nginx.conf.j2 (фрагмент)
events {}
http {
  server {
    listen {{ nginx_listen_port }};
    location / {
      return 200 "ok\n";
    }
  }
}
```

---

## ansible.cfg (настройки проекта)

`ansible.cfg` задаёт поведение Ansible по умолчанию: где искать inventory и роли, сколько параллельности (`forks`), таймауты SSH, логирование, callbacks и т.д. В production **лучше держать `ansible.cfg` в репозитории**, чтобы команда всегда выполняла playbooks с одинаковыми настройками.

### Где Ansible ищет ansible.cfg (упрощённо)

Ansible подхватывает конфиг по приоритету (сверху — выше):

- переменная окружения `ANSIBLE_CONFIG` (явный путь)
- `./ansible.cfg` в текущей директории проекта
- `~/.ansible.cfg`
- `/etc/ansible/ansible.cfg`

Production best practice: в CI задавать `ANSIBLE_CONFIG` на файл проекта, чтобы избежать влияния настроек машины runner’а.

### Пример минимального `ansible.cfg` для проекта

```ini
[defaults]
inventory = inventories/prod/hosts.ini
roles_path = roles
collections_paths = collections
forks = 20
timeout = 30
host_key_checking = True
retry_files_enabled = False
stdout_callback = yaml
callbacks_enabled = profile_tasks
log_path = ./ansible.log

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False

[ssh_connection]
pipelining = True
```

Комментарии по ключевым параметрам:

- **inventory**: путь по умолчанию (часто лучше передавать `-i` явно, а в cfg держать только дефолт для локальных прогонов).
- **roles_path / collections_paths**: фиксируют, где искать роли/коллекции — меньше «магии» и сюрпризов.
- **forks**: параллельность по хостам. В production увеличивать осторожно (можно перегрузить LB/DB).
- **timeout**: SSH timeout; слишком маленький → flaky, слишком большой → долгие фейлы.
- **host_key_checking**: в production лучше `True` (безопаснее). Для ephemeral окружений можно временно выключать, но осознанно.
- **stdout_callback / callbacks_enabled**: удобный формат вывода и профилирование задач.
- **log_path**: логирование; не писать туда, где логи попадут в публичные артефакты CI.
- **pipelining**: ускоряет SSH (уменьшает число round-trips), но требует корректного sudo без лишних TTY‑ограничений.

---

## Handlers

Handlers — специальные задачи, которые выполняются **только если** что‑то изменилось и их вызвали через `notify`.

```yaml
handlers:
  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```

### Production best practices

- Использовать handlers для reload/restart, чтобы **не перезапускать сервис без необходимости**.
- Для тяжёлых сервисов — предпочитать `reload` вместо `restart`, если возможно.

---

## Tags

Теги позволяют запускать только часть playbook.

```yaml
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
  tags: [nginx, packages]
```

```bash
# Запуск только тегов nginx
ansible-playbook -i inventories/prod/hosts.ini site.yml --tags nginx
```

### Production best practices

- Набор стандартных тегов: `packages`, `config`, `service`, `users`, `ssh`.
- В CI можно прогонять отдельные теги для быстрых проверок.

---

## Практика (мини‑кейсы)

### 1) Установить Nginx + конфиг через template

- Inventory группа `web`
- `apt` + `template` + handler `reload`
- Проверка health: `curl http://<host>:<port>`

### 2) Развернуть PostgreSQL с параметрами

Важно: в production PostgreSQL часто разворачивают через роль/коллекцию (например, `geerlingguy.postgresql`) или через managed‑сервис. Мини‑пример ниже — концептуально про переменные.

```yaml
- name: Configure postgresql.conf
  ansible.builtin.template:
    src: postgresql.conf.j2
    dest: /etc/postgresql/15/main/postgresql.conf
  notify: Restart postgresql
  tags: [postgres, config]
```

### 3) Настроить пользователей и SSH

```yaml
- name: Ensure ops user exists
  ansible.builtin.user:
    name: ops
    groups: sudo
    append: true
    create_home: true
  tags: [users]

- name: Add authorized key for ops
  ansible.builtin.authorized_key:
    user: ops
    key: "{{ lookup('file', 'files/ops_id_rsa.pub') }}"
  tags: [ssh]
```

---

## Общие production best practices

- **ansible.cfg в репозитории** (таймауты, forks, callbacks), единые настройки проекта.
- **CI‑проверки**: `ansible-lint`, `yamllint`, `ansible-playbook --syntax-check`.
- **Роли/коллекции**: повторное использование, меньше копипасты, единый стиль.
- **Секреты**: Ansible Vault или внешнее хранилище; не логировать чувствительные данные (осторожнее с `-vvv`).

