# 5. Работа с инфраструктурой: dynamic inventory, облака, rolling updates

---
Как автоматизировать реальные инфраструктуры: использовать dynamic inventory, работать с облаками (AWS/GCP/Azure), применять delegation (`delegate_to`), выполнять rolling updates через `serial`, строить zero‑downtime деплой по стратегии. В разделе — небольшие примеры кода с комментариями и production best practices.

---

## Dynamic Inventory

**Dynamic inventory** строит список хостов и групп из внешнего источника (API облака, CMDB). Это решает проблему, когда сервера создаются/удаляются автоматически и статический `hosts.ini` устаревает.

### Подходы

- **Inventory plugins** (предпочтительно): встроенные/коллекционные плагины (AWS EC2, Azure, GCP).
- **Скрипт inventory**: свой генератор JSON (устаревающий подход, но встречается).

Production best practices:

- группировать хосты по **тегам** (env, role, app, owner)
- фиксировать конфигурацию inventory‑плагина в репозитории
- ограничивать права cloud‑аккаунта (минимальные permissions только на read)

---

## AWS: EC2 inventory plugin (пример)

Требуется коллекция `amazon.aws` и доступ к AWS API (через IAM role/keys).

Файл `inventories/prod/aws_ec2.yml`:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - eu-central-1
filters:
  tag:Environment: prod
keyed_groups:
  # Группы формируются по тегам, удобно для hosts: web, db и т.п.
  - key: tags.Role
    prefix: role
compose:
  # Установить ansible_host в public_ip_address или private_ip_address
  ansible_host: private_ip_address
```

Запуск:

```bash
ansible-inventory -i inventories/prod/aws_ec2.yml --graph
ansible-playbook -i inventories/prod/aws_ec2.yml site.yml
```

Комментарий: в production чаще используют **private IP** и ходят через bastion/VPN; доступ к AWS лучше через **IAM role** (например, на runner/бастионе), а не через статические ключи.

---

## GCP / Azure (концептуально)

Идея одинаковая:

- inventory‑плагин получает список VM по API
- группирует по labels/tags
- задаёт `ansible_host` (обычно private IP)

Best practices:

- одинаковая модель тегов/labels между облаками (env/role/app)
- единый интерфейс выбора окружения: `inventories/dev`, `inventories/prod`

---

## Delegation (delegate_to)

`delegate_to` выполняет задачу **не на целевом хосте**, а на другом. Частые кейсы:

- выполнить действие на **контрол‑ноде** (localhost): запрос в API, запись артефакта, генерация конфига
- выполнить задачу на **балансировщике** (снять/вернуть ноду из пула) перед деплоем

```yaml
- name: Drain node from load balancer (example API call)
  ansible.builtin.uri:
    url: "https://lb-api.example.com/pool/remove?host={{ inventory_hostname }}"
    method: POST
    status_code: 200
  delegate_to: localhost

- name: Deploy app on host
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
  notify: Restart myapp
```

Production best practices:

- явно документировать, где выполняется задача (delegate_to), чтобы избежать сюрпризов
- для задач с внешними API — таймауты, retries и обработка ошибок

---

## Serial / rolling updates

`serial` ограничивает число хостов, обрабатываемых одновременно, и позволяет делать **rolling update**.

```yaml
- name: Rolling deploy myapp
  hosts: role_web
  become: true
  serial: "20%"   # или 2, 5 — фиксированным числом
  tasks:
    - name: Deploy config
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf
      notify: Restart myapp

    - name: Ensure service is healthy
      ansible.builtin.uri:
        url: "http://127.0.0.1:8080/health"
        method: GET
        status_code: 200
      register: health
      retries: 10
      delay: 3
      until: health.status == 200

  handlers:
    - name: Restart myapp
      ansible.builtin.service:
        name: myapp
        state: restarted
```

Комментарий: `serial` сам по себе не гарантирует zero‑downtime, но снижает риск «выключить всё сразу». Zero‑downtime достигается дополнительно через балансировщик, healthchecks и корректную стратегию.

---

## Практика: zero‑downtime деплой (идея)

Шаги:

1. **Перед деплоем** убрать хост из балансировщика (через `delegate_to: localhost` и API LB).
2. Обновить конфиг/бинарь/контейнер на хосте.
3. Перезапустить сервис (через handler).
4. Дождаться healthcheck (uri + retries/until).
5. Вернуть хост в балансировщик.
6. Повторить по `serial`.

Это можно оформить в роль (например, `app_deploy`) и переиспользовать для разных сервисов.

---

## Production best practices (сводка)

- **Dynamic inventory** для облаков, группы по тегам/labels
- **serial + healthchecks** как основа rolling update
- **delegate_to** для интеграций (LB/API), с retries/timeouts
- **private networking**: управлять по private IP через bastion/VPN
- минимальные права cloud‑аккаунта для inventory (read‑only)

