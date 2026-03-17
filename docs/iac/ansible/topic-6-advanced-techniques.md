# 6. Продвинутые техники: плагины, custom modules, Jinja2 advanced, async

---
Как писать гибкие и мощные automation‑решения: custom modules (Python), плагины (lookup/filter), продвинутый Jinja2 (фильтры, макросы), async tasks (poll), strategy plugins. В разделе — небольшие примеры кода с комментариями и production best practices.

---

## Custom modules (Python) — когда и зачем

Custom module имеет смысл, когда:

- стандартные модули не покрывают задачу (нестандартный API/CLI)
- хочется идемпотентности и предсказуемого `changed`
- shell‑скрипты стали слишком сложными и хрупкими

**Мини‑пример** (скелет модуля) — показывает структуру, без реальной логики:

```python
# library/my_custom_module.py
from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec={
            "name": {"type": "str", "required": True},
            "state": {"type": "str", "choices": ["present", "absent"], "default": "present"},
        },
        supports_check_mode=True,
    )

    name = module.params["name"]
    state = module.params["state"]

    # Здесь должна быть идемпотентная логика: определить текущее состояние и сравнить с желаемым.
    changed = False

    module.exit_json(changed=changed, name=name, state=state)

if __name__ == "__main__":
    main()
```

Использование:

```yaml
- name: Use custom module
  my_custom_module:
    name: example
    state: present
```

Production best practices:

- обязательно поддерживать `check_mode` и корректный `changed`
- писать обработку ошибок с понятными сообщениями (`module.fail_json`)
- предпочитать модули, а не обёртки над shell, если задача повторяется часто

---

## Plugins: lookup и filter

### Lookup plugins (добывать данные)

Lookup‑плагины используются в выражениях Jinja, например, чтобы получить значение из внешнего источника (файл, env, API).

```yaml
- name: Read value from env
  ansible.builtin.debug:
    msg: "token={{ lookup('env', 'MY_TOKEN') }}"
  no_log: true
```

Production best practices:

- секреты, полученные через lookup, всегда защищать `no_log: true`
- избегать «скрытой магии»: документировать источники данных (откуда берётся секрет)

### Filter plugins (преобразовывать данные)

Фильтры позволяют выносить сложную логику преобразования данных из playbook’ов в код.

#### Практика: написать свой filter plugin

Файл `filter_plugins/netmask.py`:

```python
def cidr_to_netmask(cidr: int) -> str:
    if cidr < 0 or cidr > 32:
        raise ValueError("cidr must be between 0 and 32")
    mask = (0xffffffff << (32 - cidr)) & 0xffffffff if cidr != 0 else 0
    return ".".join(str((mask >> shift) & 255) for shift in (24, 16, 8, 0))

class FilterModule(object):
    def filters(self):
        return {"cidr_to_netmask": cidr_to_netmask}
```

Использование в шаблоне или задаче:

```yaml
- name: Render interface config
  ansible.builtin.template:
    src: iface.conf.j2
    dest: /etc/network/interfaces.d/eth0.conf
```

```jinja2
{# iface.conf.j2 #}
address {{ ip_address }}
netmask {{ cidr | cidr_to_netmask }}
```

Production best practices:

- держать плагины маленькими и тестируемыми (простые unit‑тесты на Python)
- валидировать входные данные и выдавать понятные ошибки

---

## Jinja2 advanced: filters и macros

### Полезные фильтры

- `default()` — задавать значение по умолчанию
- `to_nice_yaml` / `to_json` — красиво рендерить структуры
- `regex_replace` — трансформации строк (аккуратно)

```jinja2
{# пример: макрос для генерации upstream #}
{% macro upstream(name, servers) -%}
upstream {{ name }} {
{% for s in servers %}
  server {{ s.host }}:{{ s.port }};
{% endfor %}
}
{%- endmacro %}

{{ upstream('app', backend_servers) }}
```

Production best practices:

- сложные шаблоны дробить на макросы и include’ы
- не превращать Jinja в полноценный язык программирования — лучше вынести в filter plugin

---

## Async tasks / poll

Async позволяет запускать долгие операции параллельно (например, рестарт сервисов на пачке хостов), не блокируя выполнение на каждом хосте.

```yaml
- name: Restart service asynchronously
  ansible.builtin.service:
    name: myapp
    state: restarted
  async: 120
  poll: 0
  register: restart_job

- name: Wait for restart to complete
  ansible.builtin.async_status:
    jid: "{{ restart_job.ansible_job_id }}"
  register: restart_result
  until: restart_result.finished
  retries: 30
  delay: 2
```

Production best practices:

- ограничивать параллелизм (forks/serial), иначе можно «положить» зависимость (LB/DB)
- обязательно проверять health после async‑операций

---

## Strategy plugins (концептуально)

Стратегия определяет, как Ansible планирует задачи по хостам. Стандартные:

- `linear` (по умолчанию) — синхронно по задачам
- `free` — хосты выполняются независимо (быстрее, но сложнее дебажить)

Production best practices:

- начинать с `linear` + `serial`
- `free` использовать осторожно, когда задачи независимы и важно время выполнения

---

## Практика: async deployment (рестарт сервисов)

Сценарий:

1. rolling (serial) по хостам
2. рестарт сервиса асинхронно
3. ожидание завершения + healthcheck

Это снижает общее время деплоя, не выключая весь флот одновременно.


