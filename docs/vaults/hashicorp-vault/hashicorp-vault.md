# HashiCorp Vault

![HashiCorp[200]](./images/welcome.png)
**HashiCorp Vault** — платформа **secrets management**: централизованное хранение и выдача секретов, static (например **KV**) и dynamic (БД, облака), с **политиками**, **аутентификацией**, **audit** и production‑режимом (Raft, TLS, seal/unseal).

## Материалы

- [**1. База**](topic-1-basis.md) — что такое secrets management и Vault, типы секретов, архитектура (storage, seal, tokens, policies), практика KV и ограниченный token, production best practices.
- [**2. Архитектура и развёртывание**](topic-2-architecture-and-deployment.md) — standalone и HA, Raft и Consul, Shamir и auto-unseal (AWS/GCP/Azure), TLS, бэкапы и DR, примеры `vault.hcl` и операций Raft.
- [**3. Аутентификация**](topic-3-authentication.md) — Token, AppRole, Kubernetes Auth, LDAP/OIDC; сценарии CI/CD, Pod и SSO; примеры `vault write` и логина.
- [**4. Secrets engines**](topic-4-secrets-engines.md) — KV v2 (версии, soft delete), Database и AWS, PKI (root/intermediate/issue), Transit (encrypt/sign); примеры команд.
- [**5. Интеграции**](topic-5-integrations.md) — Kubernetes (Injector, sidecar, CSI), CI/CD (GitLab CI, GitHub Actions), Vault Agent и шаблоны; env vs файлы.
- [**6. Безопасность и аудит**](topic-6-security-and-audit.md) — audit (file, syslog), least privilege, TTL и ротация, угрозы и root token; примеры политики и команд.
- [**7. Эксплуатация**](topic-7-operations.md) — Raft snapshot/restore, масштабирование, Prometheus и алерты, troubleshooting (seal, TTL, auth); практики скрейпа и падения узла.
- [**8. Продвинутые темы**](topic-8-advanced.md) — namespaces (Enterprise), performance и DR replication, HSM (PKCS#11 seal), FIPS; OSS vs Enterprise, примеры CLI и `vault.hcl`.
- [**9. Итоговый практический проект**](topic-9-capstone-project.md) — сквозная система: HA Raft, Kubernetes, CI/CD, Kubernetes auth, dynamic PostgreSQL, injection в Pod, auto-unseal, audit, бэкапы и мониторинг.

