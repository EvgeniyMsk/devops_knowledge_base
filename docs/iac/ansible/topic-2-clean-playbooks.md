# 2. Поддерживаемые playbooks: идемпотентность, условия и обработка ошибок

---
Понимать идемпотентность, уверенно использовать `register/when/failed_when/changed_when`, циклы, `block/rescue/always`, отличать `include_*` от `import_*`, работать с фактами (facts, module `setup`). В разделе — небольшие примеры кода с комментариями и production best practices.

---

## Idempotency (почему это критично)

**Idempotency** означает: повторный запуск playbook приводит систему в то же желаемое состояние **без лишних изменений**.

Почему это важно в production:

- безопасные повторные прогоны (re-run) после частичного падения
- корректный rolling update (одни и те же шаги на разных нодах)
- минимизация «шума» (перезапуски сервисов/перекаты конфигов без причин)

### Пример: «плохой» и «хороший» подход

```yaml
# Плохо: shell не гарантирует идемпотентность и часто всегда будет marked changed
- name: Add line to file (bad)
  ansible.builtin.shell: "echo 'foo=bar' >> /etc/myapp.env"

# Хорошо: модуль lineinfile гарантирует состояние
- name: Ensure foo=bar in env file (good)
  ansible.builtin.lineinfile:
    path: /etc/myapp.env
    regexp: '^foo='
    line: 'foo=bar'
    create: true
  notify: Restart myapp
```

Production best practices:

- использовать модули вместо `shell/command`
- всегда стараться, чтобы задача была **idempotent** (и чтобы это отражалось в `changed`)
- рестартовать сервисы через **handlers** (только когда реально изменилось)

---

## register / when / failed_when / changed_when

### register + when

```yaml
- name: Check if nginx is installed (dpkg)
  ansible.builtin.command: "dpkg -s nginx"
  register: nginx_check
  changed_when: false
  failed_when: false

- name: Install nginx if missing
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true
  when: nginx_check.rc != 0
```

- `changed_when: false` — проверка не должна «портить» отчёт изменений
- `failed_when: false` — проверка может вернуть non-zero, но это не ошибка playbook

### failed_when / changed_when по содержимому stdout

```yaml
- name: Run a healthcheck script
  ansible.builtin.command: "/usr/local/bin/healthcheck"
  register: hc
  changed_when: false
  failed_when: "'CRITICAL' in hc.stdout"
```

Production best practices:

- не превращать всё в shell; если нужно — аккуратно контролировать `changed_when/failed_when`
- стараться, чтобы логика ошибок была **явной и читаемой**

---

## Обработка ошибок: retries/until, ignore_errors и «остановить всё»

В production важно различать:

- **временные сбои** (сеть, репозиторий, зависимость ещё поднимается) — лечатся `retries/until`
- **ожидаемые неуспехи** (проверка «установлено ли» или «есть ли файл») — лечатся `failed_when: false`
- **настоящие ошибки** — должны останавливать прогон или хотя бы плей на хосте

### retries/until вместо sleep

```yaml
- name: Wait for service to become healthy
  ansible.builtin.uri:
    url: "http://127.0.0.1:8080/health"
    method: GET
    status_code: 200
  register: health
  retries: 10
  delay: 3
  until: health.status == 200
```

### ignore_errors (использовать осторожно)

`ignore_errors: true` позволяет продолжить выполнение даже при ошибке. Это удобно для «best effort» действий, но может скрыть реальную проблему.

```yaml
- name: Try to stop old service (best effort)
  ansible.builtin.service:
    name: legacy-service
    state: stopped
  ignore_errors: true
```

Production best practices:

- не ставить `ignore_errors` «чтобы пайплайн зелёный» — лучше настроить корректный `failed_when`
- если ошибка допустима, логика должна быть явной (например, `when:` + проверка наличия сервиса)

### assert (явно валидировать предпосылки)

```yaml
- name: Validate required vars
  ansible.builtin.assert:
    that:
      - myapp_env in ['dev', 'staging', 'prod']
      - myapp_port | int > 0
    fail_msg: "Invalid configuration for myapp"
```

### any_errors_fatal и max_fail_percentage

Когда деплой должен остановиться при проблеме на одном хосте (чтобы не «размазать» поломанную версию по флоту):

```yaml
- name: Deploy critical change
  hosts: web
  serial: "20%"
  any_errors_fatal: true
  tasks:
    - name: Deploy config
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf
```

А если допустим частичный отказ (например, несколько хостов могут быть временно недоступны), можно ограничить процент:

```yaml
- name: Deploy with tolerated failures
  hosts: web
  serial: 10
  max_fail_percentage: 10
  tasks:
    - name: Ensure package present
      ansible.builtin.apt:
        name: curl
        state: present
```

---

## Loop'ы (loop, with_items, with_dict)

`loop` — современный базовый способ.

```yaml
- name: Install common packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - curl
    - jq
    - ca-certificates
```

Цикл по словарю:

```yaml
- name: Create app directories
  ansible.builtin.file:
    path: "{{ item.value.path }}"
    state: directory
    owner: "{{ item.value.owner }}"
    mode: "{{ item.value.mode }}"
  loop: "{{ (app_dirs | dict2items) }}"
  vars:
    app_dirs:
      logs: { path: /var/log/myapp, owner: app, mode: "0755" }
      data: { path: /var/lib/myapp, owner: app, mode: "0750" }
```

Production best practices:

- избегать вложенных циклов и сложных Jinja‑выражений в одной задаче — лучше вынести в vars
- использовать `label` в loop (если вывод становится нечитаемым)

---

## Blocks (block/rescue/always)

`block/rescue/always` — структурированная обработка ошибок и «rollback» шагов.

### Практика: fallback при установке пакета

```yaml
- name: Install nginx with fallback repo
  become: true
  block:
    - name: Install nginx from default repos
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
  rescue:
    - name: Add fallback repository (example)
      ansible.builtin.apt_repository:
        repo: "ppa:nginx/stable"
        state: present

    - name: Install nginx from fallback repo
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
  always:
    - name: Ensure nginx service enabled
      ansible.builtin.service:
        name: nginx
        enabled: true
```

### Практика: «rollback» через rescue

```yaml
- name: Deploy config with rollback
  become: true
  block:
    - name: Backup current config
      ansible.builtin.copy:
        src: /etc/myapp/config.yaml
        dest: /etc/myapp/config.yaml.bak
        remote_src: true
      changed_when: false
      failed_when: false

    - name: Deploy new config
      ansible.builtin.template:
        src: config.yaml.j2
        dest: /etc/myapp/config.yaml
        mode: "0644"
      notify: Restart myapp
  rescue:
    - name: Restore previous config
      ansible.builtin.copy:
        src: /etc/myapp/config.yaml.bak
        dest: /etc/myapp/config.yaml
        remote_src: true
      notify: Restart myapp
```

Production best practices:

- `block` помогает держать «сценарий» в одном месте и не размазывать `when` по всему файлу
- «rollback» в Ansible часто ограничен (особенно для миграций), поэтому для критичных изменений нужно иметь отдельный plan B (например, фича‑флаги, быстрое переключение)

---

## Includes vs Imports

Ключевая разница:

- `import_tasks` / `import_role` — **статически**: Ansible «раскрывает» задачи на этапе парсинга
- `include_tasks` / `include_role` — **динамически**: подключение происходит во время выполнения (можно условно включать)

### Пример: условная логика по OS через include_tasks

```yaml
- name: OS-specific tasks
  ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"
```

Файлы:

```text
tasks/debian.yml
tasks/redhat.yml
```

Production best practices:

- для простых случаев предпочитать `import_tasks` (предсказуемее)
- для OS‑ветвлений и условных веток — `include_tasks`

---

## Факты (facts, setup module)

Факты (`ansible_facts`) позволяют писать кросс‑платформенные playbooks.

```yaml
- name: Gather facts (явно)
  ansible.builtin.setup:

- name: Print OS info
  ansible.builtin.debug:
    msg: "OS={{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
```

Production best practices:

- если playbook очень большой и факты не нужны — можно отключать gather_facts (ускорение), но тогда не использовать `ansible_*` факты
- не «хардкодить» пути и имена сервисов — использовать ветвления по `ansible_os_family`, `ansible_service_mgr` и т.д.

---

## Практика (итоговые задания)

- **Обработать ошибки:** реализовать fallback при установке пакета через `block/rescue/always`
- **Условная логика для разных OS:** `include_tasks` на основе `ansible_os_family`
- **Rollback:** резервная копия файла + восстановление в `rescue`

---

## Production best practices (сводка)

- **Читаемость важнее «умного Jinja»**: лучше чуть больше файлов, но проще сопровождение
- **Idempotency и handlers**: никаких лишних рестартов
- **Минимум shell**: модули → корректный diff/changed и меньший риск
- **Теги** для безопасных частичных прогонов (`--tags config`, `--tags users`)
- **Диагностика**: `changed_when/failed_when` делают отчёты предсказуемыми (и CI надёжнее)

