# 9. Best Practices: production‑grade Ansible (Middle+)

---
Как писать production‑grade Ansible‑код: балансировать DRY и читаемость, минимизировать «магию», обеспечивать предсказуемость выполнения, разделять логику и данные. В конце — список типичных антипаттернов и способы их исправления. В разделе — небольшие примеры кода с комментариями.

---

## DRY vs читаемость

DRY (Don’t Repeat Yourself) полезен, но в Ansible легко перейти грань и сделать код «магическим» и нечитаемым.

### Хороший DRY

- повторяемые шаги вынесены в **роль**
- повторяемые значения вынесены в **vars** с понятными именами

```yaml
# Хорошо: переиспользуемая роль
- hosts: web
  roles:
    - nginx
    - app
```

### Плохой DRY (магия ради DRY)

```yaml
# Плохо: сложная Jinja-логика прямо в task (трудно читать и дебажить)
- name: Render configs
  ansible.builtin.template:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
  loop: "{{ lookup('vars', env ~ '_templates') | dict2items }}"
```

Если так нужно — лучше вынести структуры данных в `group_vars`, а преобразования — в filter plugin (см. раздел 6).

---

## Минимизация magic

Под magic обычно попадает:

- скрытые зависимости между ролями (роль A «ожидает», что роль B уже поставила пакет)
- неявные значения переменных
- динамические include без ясной структуры

Production best practices:

- у ролей должен быть **README** (что делает, какие vars нужны)
- зависимости фиксировать в `meta/main.yml`
- defaults должны быть безопасными (не ломать системы «из коробки»)

---

## Предсказуемость выполнения

### Идемпотентность и handlers

Основное правило: playbook должен давать корректный `changed` и не делать лишних restarts.

```yaml
- name: Deploy config
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
  notify: Restart myapp

handlers:
  - name: Restart myapp
    ansible.builtin.service:
      name: myapp
      state: restarted
```

### Rolling updates

Для production‑сервисов применять `serial` и healthchecks, чтобы не «выключить всё сразу».

```yaml
- hosts: web
  serial: "20%"
  tasks:
    - name: Deploy config
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf
      notify: Restart myapp
```

---

## Разделение логики и данных

Логика — в ролях/задачах, данные — в `inventories/*/group_vars` и `host_vars`.

```text
roles/app/tasks/main.yml          # логика
inventories/prod/group_vars/all.yml # данные окружения (не секреты)
inventories/prod/group_vars/all/vault.yml # секреты (Vault)
```

Production best practices:

- окружения отличаются **данными**, а не копипастой playbook’ов
- секреты хранить отдельно (Ansible Vault / внешнее хранилище), задачи с секретами — `no_log: true`

---

## Антипаттерны и как их избегать

### 1) Giant playbooks (1000+ строк)

Проблема: трудно ревьюить и сопровождать, нет переиспользования.

Решение:

- разнести на роли (`common`, `nginx`, `app`, `monitoring`)
- собрать в `site.yml` (см. раздел 3)

### 2) Хардкод

Проблема: нельзя переиспользовать между окружениями.

Решение: вынести в vars.

```yaml
# Плохо
dest: /etc/myapp/prod.conf

# Хорошо
dest: "{{ myapp_config_path }}"
```

### 3) Отсутствие idempotency

Проблема: повторный прогон ломает систему или даёт ложный changed.

Решение: использовать модули и корректный `changed_when`.

### 4) shell/command вместо модулей

Проблема: меньше идемпотентности и переносимости.

Решение: использовать модули или писать custom module.

```yaml
# Плохо
- ansible.builtin.shell: useradd ops

# Хорошо
- ansible.builtin.user:
    name: ops
    state: present
```

---

## Production best practices (чеклист)

- [ ] Роли маленькие и сфокусированные (KISS)
- [ ] Минимум `shell/command`
- [ ] Идемпотентность: повторный прогон не меняет систему без необходимости
- [ ] Handlers для restart/reload
- [ ] Данные окружений в `inventories/*/group_vars`
- [ ] Секреты: Vault IDs, `no_log`, внешние secret stores при необходимости
- [ ] CI: ansible-lint + syntax-check, тесты Molecule для ключевых ролей

