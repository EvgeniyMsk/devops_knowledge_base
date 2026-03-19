# Argo CD
![ArgoCD[200]](./images/welcome.webp)
Argo CD — GitOps инструмент для управления Kubernetes приложениями через сравнение desired state (Git) и actual state (кластера). Он выполняет reconcile loop, показывает drift (например, `OutOfSync`) и позволяет безопасно синхронизировать изменения из репозитория.


## Материалы

- [**1. База**](topic-1-basis.md) — желаемое/фактическое состояние, drift detection, `pull` vs `push`, sync и reconciliation.
- [**2. Архитектура**](topic-2-architecture.md) — компоненты Argo CD (API Server, Repo Server, Application Controller), как они взаимодействуют и как идёт sync.
- [**3. Основные сущности**](topic-3-main-entities.md) — Application и AppProject, auto-sync/self-heal/prune, Sync waves.
- [**4. Sync стратегии**](topic-4-sync-strategies.md) — manual/auto sync, hooks (PreSync/PostSync) и health/rollback сценарии.
- [**5. Безопасность и доступы**](topic-5-security-access.md) — RBAC, ограничения AppProject, OIDC и управление секретами.
- [**6. Multi-cluster и multi-env**](topic-6-multi-cluster-multi-env.md) — подключение кластеров, изоляция dev/stage/prod и подходы к управлению окружениями (включая AppSet).
- [**7. Работа с Helm и Kustomize**](topic-7-helm-kustomize.md) — значения и overrides, overlays/patches, private repos и практика настройки под окружения.
- [**8. Production use cases**](topic-8-production-use-cases.md) — canary/blue-green, feature/preview окружения и практические шаблоны.
- [**9. Наблюдаемость и troubleshooting**](topic-9-observability-troubleshooting.md) — логи Argo CD, Kubernetes Events, diff view и воспроизводимые шаги восстановления.
- [**10. Архитектура репозиториев GitOps**](topic-10-gitops-repository-architecture.md) — monorepo vs multirepo, App-of-apps и guardrails через per-app периметр.
- [**11. Интеграции**](topic-11-integrations-ci-webhooks-and-notifications.md) — CI → Argo CD webhooks и уведомления Slack/Email с production-подходом.
- [**12. Advanced (для уверенного Middle+)**](topic-12-advanced-image-updater-health-exclusions-performance-tuning.md) — Image Updater, custom health checks, resource exclusions и performance tuning.