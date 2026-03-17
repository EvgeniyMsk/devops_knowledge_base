# 4. Безопасность и секреты: Ansible Vault, Vault IDs, no_log

---
Как сделать automation production‑ready: безопасно хранить и использовать секреты, разделять секреты по окружениям (dev/prod), понимать Ansible Vault и Vault IDs, применять `no_log` и избегать утечек в логах и артефактах CI.

---

## Ansible Vault: базовые принципы

**Ansible Vault** шифрует файлы/строки с секретами (пароли, токены, приватные ключи) и позволяет хранить их в Git **в зашифрованном виде**.

Что важно в production:

- Vault защищает **в покое** (at rest) — но ключ/пароль к Vault должен храниться безопасно (Vault password file в секрет‑хранилище, переменные CI, 1Password и т.п.).
- Разделять секреты по окружениям (dev/staging/prod) и по доменам (DB, API, monitoring).
- Минимизировать попадание секретов в stdout/stderr и артефакты CI.

---

## Vault IDs

**Vault ID** — механизм, позволяющий использовать **несколько ключей Vault** в одном проекте (например, отдельный ключ для dev и для prod). Это удобно, когда:

- разные команды имеют доступ только к dev
- разные окружения должны расшифровываться разными ключами
- вы хотите ротацию ключей по частям

### Пример структуры (production‑style)

```text
inventories/
  dev/
    hosts.ini
    group_vars/
      all/
        vault.yml          # зашифровано vault id: dev
  prod/
    hosts.ini
    group_vars/
      all/
        vault.yml          # зашифровано vault id: prod
```

### Запуск с Vault ID

```bash
# Вариант 1: пароль вводится интерактивно
ansible-playbook -i inventories/prod/hosts.ini site.yml --vault-id prod@prompt

# Вариант 2: пароль берётся из файла (например, CI secret file, chmod 600)
ansible-playbook -i inventories/prod/hosts.ini site.yml --vault-id prod@/run/secrets/ansible_vault_prod

# Для dev:
ansible-playbook -i inventories/dev/hosts.ini site.yml --vault-id dev@prompt
```

Production best practices:

- **Не хранить** vault password file в репозитории.
- В CI — использовать секрет‑переменные/secret files, доступные только нужным веткам/окружениям.
- Использовать разные Vault IDs для dev и prod.

---

## Хранение секретов: Vault vs внешние системы

Реальные варианты:

- **Ansible Vault** — простой и удобный, когда секреты нужны только для конфигурации и доступны через GitOps‑подход.
- **Внешние системы** (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) — лучше, когда нужна централизованная ротация, аудит, short‑lived креды и интеграции.

Типичный production‑подход:

- в Git хранить **минимум секретов** (или хранить только зашифрованные Ansible Vault)
- для «тяжёлых» секретов и ротации — брать значения из внешнего хранилища на этапе запуска playbook (lookup‑плагины/скрипты), либо подгружать секреты в CI

---

## no_log: защита от утечек в логах

`no_log: true` скрывает вывод задачи (включая параметры) в логах Ansible. Это must‑have для задач, которые оперируют паролями/токенами.

```yaml
- name: Create DB user (do not leak password)
  community.postgresql.postgresql_user:
    name: app_user
    password: "{{ db_password }}"
    state: present
  no_log: true
```

Важно:

- `no_log` скрывает вывод задачи, но **не решает** проблему неправильного хранения секретов.
- Осторожно с `debug: var=...` и `-vvv` — это частая причина утечек.

Production best practices:

- помечать все чувствительные задачи `no_log: true`
- избегать вывода секретов даже в masked‑переменные CI (маскирование не всегда надёжно)

---

## Практика: зашифровать пароли БД

### Пример: vault.yml (зашифрован)

```yaml
# group_vars/all/vault.yml (этот файл должен быть зашифрован vault'ом)
db_user: app_user
db_password: "S3cureP@ssw0rd"
```

Команды:

```bash
# Создать зашифрованный файл
ansible-vault create inventories/prod/group_vars/all/vault.yml --vault-id prod@prompt

# Отредактировать зашифрованный файл
ansible-vault edit inventories/prod/group_vars/all/vault.yml --vault-id prod@prompt

# Посмотреть (лучше не делать в CI)
ansible-vault view inventories/prod/group_vars/all/vault.yml --vault-id prod@prompt
```

Использование в задаче:

```yaml
- name: Create DB user (password from vault)
  community.postgresql.postgresql_user:
    name: "{{ db_user }}"
    password: "{{ db_password }}"
    state: present
  no_log: true
```

---

## Практика: разделить vault по окружениям (dev/prod)

Подход:

- `inventories/dev/group_vars/all/vault.yml` — dev секреты (Vault ID `dev`)
- `inventories/prod/group_vars/all/vault.yml` — prod секреты (Vault ID `prod`)
- playbooks остаются одинаковыми, отличия — только в inventory/vars

```bash
ansible-playbook -i inventories/dev/hosts.ini site.yml --vault-id dev@prompt
ansible-playbook -i inventories/prod/hosts.ini site.yml --vault-id prod@prompt
```

---

## Production best practices (чеклист)

- [ ] Секреты не хранятся в plaintext в Git
- [ ] Для окружений разные Vault IDs / разные ключи
- [ ] Vault password хранится в секрет‑хранилище (не в репо)
- [ ] `no_log: true` на задачах с паролями/токенами
- [ ] В CI отключён/ограничен подробный вывод, секреты не попадают в артефакты
- [ ] Есть процесс ротации секретов (и тестирование ротации на staging)

