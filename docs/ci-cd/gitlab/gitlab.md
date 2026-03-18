# GitLab CI/CD
![Gitlab](./images/welcome.jpeg)
---

В этом разделе — практики **GitLab CI/CD**: проектируем пайплайны, оптимизируем время сборки (`cache`/`artifacts`), делаем надёжные условия запуска (`rules`), управляем зависимостями (`needs`/`dependencies`) и разворачиваем в безопасном production‑режиме.

---

## Материалы

- [**1. База**](topic-1-basis.md) — структура `.gitlab-ci.yml`, stages/jobs, variables, `rules`, `needs`/`dependencies`, cache и artifacts.
- [**2. Продвинутая логика**](topic-2-advanced-logic.md) — сложные `rules`, динамические пайплайны, DAG, параллельные/матричные job’ы, child/parent.
- [**3. Архитектура платформы**](topic-3-platform-architecture.md) — монорепо vs мульти‑репо, reusable templates, “pipeline as a platform”, include/child pipelines, GitOps-подходы.
- [**4. Deployment стратегии**](topic-4-deployment-strategies.md) — environments, review apps, manual approvals, blue/green и canary, rollback и release.
- [**5. Безопасность CI/CD**](topic-5-ci-security.md) — секреты и CI/CD Variables (masked/protected), безопасность runner’ов, SAST/DAST и secret detection.
- [**6. Оптимизация и performance**](topic-6-optimization-performance.md) — ускорение пайплайнов (cache/artifacts, Docker layer caching), `rules:changes`, DAG через `needs`, матрицы.
- [**7. Интеграции**](topic-7-integrations.md) — Deploy в Kubernetes из CI, Terraform/Ansible интеграции, Auto DevOps и separation infra/app.
- [**8. Observability и отладка**](topic-8-observability-debugging.md) — job logs, pipeline metrics, работа с flaky/failure-паттернами, диагностика и артефакты.
- [**9. Реальные кейсы**](topic-9-real-cases.md) — примеры эксплуатации: масштабирование пайплайнов, миграции, снижение build time и практика triage.


---