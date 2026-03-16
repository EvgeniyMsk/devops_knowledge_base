# 6. Автоматизация и DevOps: PostgreSQL в Kubernetes, IaC, миграции

**Цель:** понимать, как запускать и автоматизировать PostgreSQL в контексте DevOps: ограничения StatefulSet и persistent volumes в Kubernetes, когда Postgres не стоит поднимать в K8s; операторы (Zalando, CrunchyData); IaC (Ansible, Terraform для managed DB) и миграции схемы (Flyway, Liquibase). В разделе — примеры кода и конфигурации с комментариями и best practices для production.

---

## 14. PostgreSQL + Docker / Kubernetes

### Почему StatefulSet ≠ HA

**StatefulSet** даёт стабильные имена подов (postgres-0, postgres-1) и отдельный PVC на каждый под — это основа для stateful-приложения, но **сам по себе не даёт HA**:

- Один StatefulSet с одной репликой — одна точка отказа; при падении пода Kubernetes пересоздаст под на той же ноде или другой, но **автоматического переключения на другую реплику (failover) нет**.
- Несколько реплик (postgres-0 primary, postgres-1 standby) требуют отдельной логики: кто primary, кто standby, кто при падении primary станет новым primary. Эту логику реализуют **операторы** (Patroni внутри оператора, CrunchyData, Zalando) или внешние системы (вне K8s).

Итого: StatefulSet + PVC — корректная упаковка одного инстанса или пары primary/standby; полноценный HA с автоматическим failover — только с оператором или внешним кластером.

### Проблемы с persistent volumes в K8s

- **Привязка к зоне/ноде:** RWO (ReadWriteOnce) том привязан к ноде; при eviction пода том должен attach к другой ноде — в облаке это возможно только в пределах зоны (EBS в той же AZ). При потере ноды восстановление пода на другой ноде может потребовать времени и доступности тома в этой зоне.
- **Размер и расширение:** PVC с фиксированным размером; расширение (expand) поддерживается не всеми StorageClass и не для всех объёмов. Заранее закладывать рост или использовать динамическое расширение где возможно.
- **Снапшоты и бэкапы:** для PITR и восстановления нужны снапшоты томов или логические бэкапы (pg_dump, pg_basebackup, wal-g) вне пода. Не полагаться только на один PVC без регулярных бэкапов.

```yaml
# Фрагмент StatefulSet для Postgres: один под с одним томом — не HA, только стабильное имя и том.
# Полноценный кластер — через оператор (см. ниже).
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
        storageClassName: fast-ssd
```

### Когда Postgres не стоит запускать в Kubernetes

- **Строгие SLA и зрелость операций:** управление репликацией, бэкапами, обновлениями и отказоустойчивостью в K8s требует либо оператора, либо значительных собственных скриптов. При недостаточной экспертизе проще использовать **managed PostgreSQL** (RDS, Cloud SQL, Azure Database, Crunchy Bridge и т.д.).
- **Тяжёлая и критичная нагрузка:** дисковая и сетевая латентность в облачных томах, ограничения по IOPS и пропускной способности. Для очень больших и критичных БД часто выбирают выделенные VM или managed-сервис с гарантированным IOPS.
- **Команда не готова к операционному циклу:** обновления K8s и нод, drain, поведение при eviction, бэкапы и PITR — всё это нужно уметь и автоматизировать. Иначе проще «база на VM + скрипты» или managed.

Итого: в K8s Postgres имеет смысл при готовности использовать оператор, настраивать бэкапы и понимать ограничения storage; иначе — managed или VM.

### Операторы: Zalando Postgres Operator, CrunchyData

**Zalando postgres-operator** — управляет кластером PostgreSQL в Kubernetes: создаёт StatefulSet’ы, сервисы, создаёт пользователей и базы через CRD, может интегрировать с Patroni для failover. Ресурс **postgresql** (CRD) описывает кластер (число реплик, образ, ресурсы, бэкапы).

**Crunchy Data PostgreSQL Operator (PGO)** — похожая идея: CRD для кластера PostgreSQL, автоматическое развёртывание primary и replicas, бэкапы (pgBackRest), мониторинг. Подходит для production-кластеров в K8s.

```yaml
# Пример CR Zalando postgres-operator (упрощённо; актуальный формат смотреть в документации оператора)
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: my-app-db
  namespace: production
spec:
  teamId: myteam
  numberOfInstances: 2
  users:
    app_user: [superuser]
  databases:
    mydb: app_user
  postgresql:
    version: "15"
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "2Gi"
  volume:
    size: 20Gi
```

Установка: Helm-чарт оператора в кластер; затем создание CR postgresql. Оператор создаёт нужные StatefulSet’ы, сервисы, секреты и при необходимости бэкапы. Перед использованием изучить документацию выбранного оператора (Zalando или CrunchyData) и требования к StorageClass и правам.

---

## 15. IaC и CI/CD

### Ansible роли для PostgreSQL

Роли Ansible позволяют описать установку и настройку PostgreSQL на VM: установка пакетов, копирование **postgresql.conf** и **pg_hba.conf**, создание ролей и баз, настройка репликации. Повторяемость и версионирование конфигурации в Git.

```yaml
# Пример задачи: шаблон конфига и уведомление handler на reload (фрагмент роли)
# tasks/main.yml
- name: Deploy postgresql.conf
  template:
    src: postgresql.conf.j2
    dest: "{{ pg_config_dir }}/postgresql.conf"
  notify: Reload PostgreSQL

# handlers/main.yml
- name: Reload PostgreSQL
  systemd:
    name: postgresql
    state: reloaded
```

Переменные (версия Postgres, shared_buffers, список баз и пользователей) выносятся в **vars** или **defaults**; секреты — в Ansible Vault или внешнее хранилище. Best practice: одна роль для установки и базовая конфигурация, отдельные роли или playbook для создания БД/пользователей под приложения.

### Terraform (managed DB)

Для **managed PostgreSQL** (AWS RDS, Google Cloud SQL, Azure Database) инфраструктура описывается в Terraform: инстанс, параметры группы, подсети, секреты. Миграции схемы и данных при этом остаются в зоне CI/CD (Flyway, Liquibase, скрипты).

```hcl
# Пример: AWS RDS PostgreSQL (фрагмент)
resource "aws_db_instance" "main" {
  identifier     = "myapp-postgres"
  engine         = "postgres"
  engine_version = "15"
  instance_class = "db.t3.medium"
  allocated_storage = 20
  storage_type  = "gp3"

  db_name  = "mydb"
  username = "admin"
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  publicly_accessible    = false
  backup_retention_period = 7
  backup_window         = "03:00-04:00"
  maintenance_window    = "sun:04:00-sun:05:00"
}
```

Terraform не заменяет миграции схемы: создание таблиц, индексов, изменений DDL делают через миграции в пайплайне приложения.

### Миграции (Flyway, Liquibase) — концептуально

**Миграции схемы** — версионированные скрипты (SQL или код), которые применяются к БД в порядке версий. При деплое приложения пайплайн запускает миграции до или после выката кода; БД переходит в нужную версию схемы.

- **Flyway** — миграции в виде SQL-файлов (или Java) с именем по версии (V1__create_orders.sql, V2__add_index.sql). Flyway хранит историю в служебной таблице; применяет только новые миграции.
- **Liquibase** — миграции в XML/YAML/SQL; изменения описываются декларативно (createTable, addColumn и т.д.). Подходит для кросс-БД и сложных сценариев.

Общие принципы для DevOps:

- Миграции **идемпотентны** там, где возможно (проверка «если объект не существует — создать»), либо строго линейны и не меняют уже применённые.
- Миграции запускаются в **CI/CD** (отдельный job до или после деплоя приложения); один источник истины в Git.
- Откат схемы — отдельная тема (Liquibase rollback, либо ручные скрипты); в production предпочитают forward-only миграции и новые миграции для отката изменений.

```bash
# Типичный вызов Flyway в CI (пример)
flyway -url=jdbc:postgresql://$DB_HOST/mydb -user=$DB_USER -password=$DB_PASSWORD -locations=filesystem:./migrations migrate
```

---

## Best practices для production

| Область | Рекомендация |
|--------|---------------|
| **Postgres в K8s** | Использовать оператор (Zalando или CrunchyData) при необходимости HA и автоматизации; не полагаться на «голый» StatefulSet как на HA. |
| **Storage** | Адекватный StorageClass (IOPS, размер), регулярные бэкапы и при необходимости снапшоты; мониторинг места на диске. |
| **IaC** | Конфигурация БД (конфиги, роли, при необходимости инстансы) в Git; Terraform для managed, Ansible для self-hosted на VM. |
| **Миграции** | Версионированные миграции в репозитории приложения; применение в пайплайне; тестирование на копии БД перед production. |

---

## Паттерны и антипаттерны

| Паттерн | Описание |
|--------|----------|
| **Оператор для Postgres в K8s** | Единая модель (CR) для кластера, бэкапы и при необходимости failover. |
| **Миграции в CI/CD** | Один источник истины схемы в Git; применение при деплое. |
| **Terraform для managed DB** | Инстанс и сеть в коде; пароли и секреты через переменные/Vault. |

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **StatefulSet с одной репликой считать HA** | Нет автоматического failover при падении пода. | Использовать оператор или внешний кластер с Patroni. |
| **Ручные изменения схемы в production** | Дрейф между окружениями, нет истории. | Все изменения через миграции в Git. |
| **Postgres в K8s без бэкапов** | Потеря PVC или кластера — потеря данных. | Настроить бэкапы (оператор, wal-g, pgBackRest) и проверять restore. |

---

## Дополнительные материалы

- [Zalando postgres-operator](https://github.com/zalando/postgres-operator)
- [Crunchy Data PostgreSQL Operator (PGO)](https://www.crunchydata.com/products/crunchy-postgresql-operator)
- [Kubernetes — 6. Хранилище](../../kubernetes/topic-6-storage.md) — PVC, StatefulSet
- [Flyway](https://flywaydb.org/)
- [Liquibase](https://www.liquibase.org/)
- [3. High Availability и масштабирование](topic-3-ha-scaling.md) — репликация, Patroni
