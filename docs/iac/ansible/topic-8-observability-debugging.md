# 8. Observability и Debugging: verbose, debug, callback plugins, profiling

---
Как поддерживать и чинить Ansible‑автоматизацию: правильно использовать verbosity (`-v/-vvv`), `debug`, callback plugins, логирование и профилирование playbooks. В конце — практики: найти/починить flaky playbook и ускорить медленный deploy. В разделе — небольшие примеры кода с комментариями и production best practices.

---

## Verbosity (`-v`, `-vv`, `-vvv`)

Уровни детализации:

- `-v` — базовая доп. информация
- `-vv` — больше деталей (включая результаты модулей)
- `-vvv` — максимально подробно (может содержать чувствительные данные при неосторожных задачах)

```bash
ansible-playbook -i inventories/prod/hosts.ini site.yml -v
ansible-playbook -i inventories/prod/hosts.ini site.yml -vv
ansible-playbook -i inventories/prod/hosts.ini site.yml -vvv
```

Production best practices:

- в CI для production избегать `-vvv` без необходимости
- на задачах с секретами всегда `no_log: true` (см. раздел 4)

---

## Debug module

`debug` помогает быстро увидеть значения переменных, ветвление `when` и результаты `register`.

```yaml
- name: Show computed values
  ansible.builtin.debug:
    msg:
      - "env={{ env }}"
      - "host={{ inventory_hostname }}"
      - "os={{ ansible_facts['os_family'] }}"
```

Для сложных структур:

```yaml
- name: Dump var
  ansible.builtin.debug:
    var: some_complex_var
```

Production best practices:

- не выводить секреты через debug
- временный debug удалять после фикса (или прятать за tag `debug` и не запускать в prod)

---

## Callback plugins

Callback plugins влияют на формат вывода и могут давать профилирование/тайминги.

### Пример: включить тайминги задач (profile_tasks)

В `ansible.cfg`:

```ini
[defaults]
callbacks_enabled = profile_tasks
```

Это добавит в вывод длительность каждой задачи — удобно для ускорения медленных плейбуков.

Production best practices:

- включать `profile_tasks` в CI (или локально) для оптимизации
- для production логов выбирать формат, который удобно парсить (например, json callback — если используете централизованное логирование)

---

## Логирование

В `ansible.cfg`:

```ini
[defaults]
log_path = ./ansible.log
```

Production best practices:

- не писать логи в общий артефакт CI, если есть риск секретов (даже с маскированием)
- ограничивать доступ к логам, хранить ретеншн
- `no_log: true` на чувствительных задачах

---

## Profiling playbooks (ускорение)

Что чаще всего замедляет:

- сбор фактов на каждом запуске (gather_facts) при большом флоте
- слишком маленький `forks`
- лишние `shell` и команды, которые можно заменить модулем
- отсутствие `serial` и неэффективный порядок задач

### Пример: ускорить за счёт forks и отключения фактов

```bash
# forks: больше параллельности (с осторожностью)
ansible-playbook -i inventories/prod/hosts.ini site.yml --forks 30
```

```yaml
- name: Fast play (facts off)
  hosts: web
  gather_facts: false
  tasks:
    - name: Ping
      ansible.builtin.ping:
```

Комментарий: `gather_facts: false` использовать только если play действительно не зависит от фактов.

---

## Практика: найти и исправить “flaky” playbook

Типовые причины flaky:

- задачи зависят от таймингов (сервис ещё не поднялся, файл ещё не появился)
- нет retries/until для внешних зависимостей
- неидемпотентные shell‑команды

### Пример: заменить sleep на retries/until

```yaml
- name: Wait for service health
  ansible.builtin.uri:
    url: "http://127.0.0.1:8080/health"
    method: GET
    status_code: 200
  register: hc
  retries: 10
  delay: 3
  until: hc.status == 200
```

---

## Практика: ускорить медленный deploy

Шаги:

1. включить `profile_tasks` и найти топ самых медленных задач
2. убрать лишний `gather_facts` (где возможно) или ограничить facts (subset)
3. увеличить `forks` и/или использовать `serial` для контролируемого rolling
4. параллелить долгие операции через async/poll (см. раздел 6)

---

## Production best practices (сводка)

- `profile_tasks` для анализа времени
- аккуратное использование `-vvv` и `debug` (не светить секреты)
- retries/until вместо sleep
- корректная параллельность (`forks`, `serial`) под лимиты инфраструктуры

