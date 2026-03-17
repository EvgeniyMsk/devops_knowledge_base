# Ansible
![Ansible[200]](./images/welcome.png)
## Разделы

- [**1. База: inventory, playbooks, переменные и шаблоны**](topic-1-basis.md) — static/dynamic inventory, структура playbook и flow выполнения, модули и idempotency, переменные (group_vars/host_vars), templates (Jinja2), handlers, tags; мини‑практика (Nginx + template, PostgreSQL, пользователи и SSH) и production best practices.
- [**2. Поддерживаемые playbooks**](topic-2-clean-playbooks.md) — идемпотентность, `register/when/failed_when/changed_when`, loop’ы, `block/rescue/always`, includes vs imports, facts; практики: fallback, OS‑логика, rollback.
- [**3. Архитектура проекта**](topic-3-project-architecture.md) — роли (структура, defaults vs vars), организация репозитория (roles/, inventories/, group_vars/), переиспользование через Ansible Galaxy/collections; практика: разбить большой playbook на роли.
- [**4. Безопасность и секреты**](topic-4-security-secrets.md) — Ansible Vault, Vault IDs, хранение секретов (Vault vs внешние системы), `no_log`; практика: шифрование паролей БД, разделение vault по окружениям.
- [**5. Работа с инфраструктурой**](topic-5-infrastructure-operations.md) — dynamic inventory, облака (AWS/GCP/Azure), `delegate_to`, `serial` и rolling updates; практика: zero‑downtime деплой.
- [**6. Продвинутые техники**](topic-6-advanced-techniques.md) — custom modules (Python), plugins (lookup/filter), Jinja2 advanced (filters/macros), async tasks/poll, strategy plugins; практика: свой filter plugin и async‑деплой.
- [**7. CI/CD и интеграция**](topic-7-cicd-integration.md) — интеграция с GitLab CI/Jenkins, проверки (ansible-lint, molecule), Test‑driven Ansible; практика: pipeline lint → syntax check → deploy и Molecule тест роли.
- [**8. Observability и Debugging**](topic-8-observability-debugging.md) — verbosity (-vvv), debug module, callback plugins, логирование, profiling playbooks; практика: flaky playbook и ускорение деплоя.
- [**9. Best Practices (Middle+)**](topic-9-best-practices.md) — DRY vs читаемость, минимизация magic, предсказуемость выполнения, разделение логики и данных; антипаттерны (giant playbooks, хардкод, отсутствие idempotency, shell вместо модулей).
