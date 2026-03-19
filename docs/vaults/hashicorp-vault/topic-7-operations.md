# 7. Эксплуатация (production readiness)

---
Цель — подготовить Vault к **production**: **бэкапы/восстановление**, **Raft snapshots**, масштабирование кластера, **наблюдаемость** (Prometheus, алерты) и **типовой troubleshooting** (seal, TTL токенов, auth). В конце — практика: скрейп метрик и симуляция падения узла.

---

## Операции: бэкап и восстановление

| Задача | Инструмент | Production-комментарий |
|--------|------------|-------------------------|
| **Снимок состояния (Raft)** | `vault operator raft snapshot` | Основной способ для **integrated storage**; хранить snapshot **вне** кластера, шифровать бэкап at-rest. |
| **Восстановление** | `snapshot restore` на **остановленных** / подготовленных узлах по runbook | Обязателен **регулярный drill** на тестовом контуре. |
| **Логическая миграция** | Репликация / миграции по гайдам HashiCorp | Не путать snapshot с клонированием «нажатием одной кнопки» между регионами без проверки лицензии и топологии. |

### Примеры snapshot

```bash
# Снять снимок (из контура с правами operator / по политике)
vault operator raft snapshot save /secure/backups/vault-$(date -u +%Y%m%dT%H%M%SZ).snap

# Проверить целостность (опционально, зависит от версии/утилит)
vault operator raft snapshot inspect /secure/backups/vault-20250315T120000Z.snap
```

```bash
# Восстановление выполняют строго по документации к вашей версии Vault,
# часто на «чистом» узле или в maintenance window:
# vault operator raft snapshot restore /path/to/file.snap
```

Комментарий: перед restore уточните, не сломает ли это **кворум**; для production держите письменный **runbook** и окно работ.

---

## Масштабирование (Raft)

| Решение | Практика |
|---------|----------|
| **Размер кластера** | Обычно **3** или **5** voter-узлов; нечётное число для кворума. |
| **Рост нагрузки** | Сначала **железо/IOPS** лидера и сетевые лимиты, затем **read replicas** / архитектурные паттерны по лицензии и документации. |
| **Добавление пира** | Через `retry_join` / API **raft** при уже работающем лидере; проверка `vault operator raft list-peers`. |

Best practice: не масштабировать «вверх» без метрик — диск Raft, latency записи и **leader step-down** должны быть на дашборде.

---

## Мониторинг: Prometheus

Vault отдаёт **метрики** (в т.ч. в формате Prometheus при запросе с нужным `format`), если включён **telemetry** и разрешён доступ к endpoint.

### Фрагмент `vault.hcl` (telemetry)

```hcl
telemetry {
  prometheus_retention_time = "30s"
  disable_hostname          = true
}

# В production доступ к /v1/sys/metrics обычно ограничивают токеном или сетью;
# открытый доступ без аутентификации допустим только осознанно (часто запрещён политикой).
```

Комментарий: точные поля (`unauthenticated_metrics_access` и др.) зависят от версии Vault — сверяйтесь с документацией **Telemetry** для вашей ветки.

### Пример `scrape_configs` для Prometheus

```yaml
scrape_configs:
  - job_name: vault
    scrape_interval: 30s
    scheme: https
    metrics_path: /v1/sys/metrics
    params:
      format: ["prometheus"]
    static_configs:
      - targets: ["vault-1.internal:8200", "vault-2.internal:8200", "vault-3.internal:8200"]
    # Файл с сырой строкой токена (без префикса Bearer) — см. версию Prometheus
    bearer_token_file: /etc/prometheus/secrets/vault-metrics.token
    tls_config:
      ca_file: /etc/prometheus/tls/ca.pem
```

Best practices:

- токен только для **чтения метрик**, отдельная политика, ротация;
- TLS verify (`tls_config`) к вашему PKI;
- не скрейпить только лидера — по узлам проще ловить **локальные** симптомы.

---

## Алертинг (ориентиры)

Примеры условий (имена метрик могут отличаться по версии — проверьте после включения скрейпа):

| Симптом | Зачем алертить |
|---------|----------------|
| `vault.core.sealed` / статус узла **sealed** | Нода не обслуживает трафик |
| Потеря кворума / нет лидера Raft | Полная недоступность записи |
| Рост задержек **storage** / ошибок Raft | Риск деградации или диска |
| Всплеск **4xx/5xx** на auth paths | Атака, массовое истечение токенов, поломка IdP |
| Место на томе **audit** и **Raft** | Риск остановки из-за «disk full» |

Интеграция: Alertmanager → on-call; **runbook** на каждый критичный алерт.

---

## Troubleshooting

### Seal: кластер или узел «запечатан»

| Признак | Что проверить |
|---------|----------------|
| `vault status` → **Sealed: true** | KMS/IAM auto-unseal, сеть до KMS, квота API |
| После рестарта все узлы sealed | Потеря доступа KMS, смена CMK без миграции |
| Один узел sealed | Локальный сбой тома / конфигурации |

Действия: runbook на **Shamir unseal** или восстановление роли KMS; не отключать audit «про запас» без анализа — сначала изоляция причины.

### Истечение токенов (token expiration)

- Симптомы: **403** с кодами вроде *permission denied* / *token expired* в зависимости от клиента.
- Проверить: `token lookup` (если есть accessor), TTL на **роли auth**, orphan vs renewable.
- Best practice: в приложениях **явное renew** только при долгоживущих процессах; для batch job — новый login на каждый запуск.

```bash
# Если ещё есть accessor (из лога аудита)
vault token lookup -accessor "$ACCESSOR"
```

### Ошибки аутентификации (auth failures)

| Причина | Проверка |
|---------|----------|
| Неверная роль Kubernetes | `bound_service_account_*`, issuer JWT, audience |
| Истёкший **secret_id** AppRole | `secret_id_num_uses`, TTL, отзыв |
| Рассинхронизация времени | NTP на узлах и в CI |
| Блокировка сети до Vault | MTLS, firewall, service mesh |

```bash
# Тест логина из контура приложения (пример Kubernetes)
vault write -format=json auth/kubernetes/login role=myapp jwt="$JWT" | jq .
```

---

## Практика 1: настроить scraping Prometheus

1. Включить **telemetry** в `vault.hcl` и перезагрузить согласно вашему способу деплоя.
2. Создать токен с политикой только на `sys/metrics` (или использовать ваш стандартный механизм auth для Prometheus).
3. Добавить job в Prometheus, включить **TLS verify**.
4. Убедиться на `/targets`, что все **три** узла в **UP** (или ваша топология).

---

## Практика 2: симулировать падение узла

**В Kubernetes (пример мысленного сценария):**

- удалить/остановить **один** follower Pod или drained ноду;
- наблюдать **лидера** и время **re-election**;
- убедиться, что приложения с retry обращаются к **service** Vault, а не к одному IP.

**На VM:**

- остановить `vault` на follower;
- `vault operator raft list-peers` с оставшегося узла — кворум сохранён при 3-of-5 и т.д.

Best practice: упражнение проводить **до** продакшена; фиксировать RTO фактическим временем.

---

## Сводный checklist эксплуатации

| # | Пункт |
|---|--------|
| 1 | Расписание snapshot + off-site хранение + тест restore |
| 2 | Дашборды: sealed, leader, latency, disk, audit |
| 3 | Алерты с runbook |
| 4 | Учения: падение узла, заполнение диска audit |
| 5 | Документированные версии Vault и процедура апгрейда |

---

## Дополнительные материалы

- [Raft snapshots](https://developer.hashicorp.com/vault/docs/commands/operator/raft#snapshot)
- [Telemetry](https://developer.hashicorp.com/vault/docs/configuration/telemetry)
- [Monitor telemetry & audit device log data](https://developer.hashicorp.com/vault/docs/internals/telemetry)

