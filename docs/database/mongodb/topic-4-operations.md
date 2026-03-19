# 4. Операционка: деплой, конфигурация и бэкапы

---
Эксплуатация MongoDB в **production** опирается на предсказуемый **деплой** (контейнеры / Kubernetes), явную **конфигурацию**, **безопасность** (внутренняя аутентификация членов, TLS) и проверенную стратегию **backup / restore** включая сценарии **частичного отказа**. Ниже — практический минимум для DevOps/SRE.

---

## Деплой: Docker и Kubernetes

| Подход | Комментарий |
|--------|-------------|
| **Docker / Compose** | Быстрый старт для dev/stage; в production — оркестратор, healthcheck, лимиты ресурсов, тома с правильным **storage class** |
| **Kubernetes** | Каждый `mongod` — **стабильное сетевое имя** и **персистентный диск**; обновления без потери кворума replica set |
| **StatefulSet** | Типичный способ: по одному поду на член RS, **PVC** на под, **Headless Service** для DNS вида `mongo-0.mongo`, `mongo-1.mongo` |

### Persistent Volumes

- **Данные** и **журнал** WiredTiger должны жить на PV, выдерживающем ваш **IOPS/latency** SLA.
- **Production:** не пересоздавайте кластер без плана миграции данных; «потерянный» PV = потеря узла без бэкапа.

Пример **идеи** манифеста (фрагмент; порты и образ под ваш стандарт):

```yaml
# Упрощённо: StatefulSet + volumeClaimTemplates для данных
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo
  replicas: 3
  template:
    spec:
      containers:
        - name: mongod
          image: mongo:7
          # args: ["--replSet", "rs0", "--bind_ip_all", "--config", "/etc/mongod.conf"]
          volumeMounts:
            - name: data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
```

**Best practice:** `rs.initiate` / добавление членов делайте **один раз** после готовности DNS; автоматизация через оператор (например **Percona**, **MongoDB Kubernetes Operator**) снижает риск ручных ошибок.

---

## Конфигурация: `mongod.conf`

Основные блоки (имена параметров см. [документацию](https://www.mongodb.com/docs/manual/reference/configuration-options/) под вашу версию):

```yaml
# /etc/mongod.conf — пример структуры
storage:
  dbPath: /data/db
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4  # Подбирать под хост; не занимать всю RAM

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

net:
  port: 27017
  bindIp: 0.0.0.0  # В production сузить + firewall
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongo.pem
    CAFile: /etc/ssl/ca.pem

security:
  authorization: enabled
  keyFile: /etc/mongo-secret/keyfile  # Члены replica set

replication:
  replSetName: rs0
```

**Production:** держите **один источник правды** для конфига (Helm values, GitOps); не правьте боевые флаги «с консоли» без учёта в репозитории.

---

## `keyFile` и аутентификация между членами replica set

- Файл с **общим секретом** на всех членах RS; права **`chmod 400`**, владелец процесса `mongod`.
- После включения **`authorization`** и **keyFile** узлы доверяют друг другу как членам одного набора.

```bash
# Лаборатория: сгенерировать keyFile
openssl rand -base64 756 > /etc/mongo-secret/keyfile
chmod 400 /etc/mongo-secret/keyfile
chown mongodb:mongodb /etc/mongo-secret/keyfile
```

**Best practice:** в Kubernetes секрет монтируется как **readOnly** volume; ротация keyFile требует **согласованной** процедуры на всех членах (план окна обслуживания).

---

## TLS

| Элемент | Замечание |
|---------|-----------|
| **В транзите** | Клиенты и члены RS — **TLS**; отключать только в изолированной dev-сети |
| **Сертификаты** | Короткоживущие от **корпоративного CA** или cert-manager в K8s |
| **Имя в SAN** | Должно совпадать с тем, как приложение подключается (DNS подов / сервисов) |

В connection string клиента часто указывают **`tls=true`** и доверенный CA; детали — по драйверу.

---

## Backup и восстановление

### `mongodump` / `mongorestore`

- **Логические** бэкапы: гибко, но на больших объёмах дольше и нагрузка на боевой кластер.
- Подходит для **отдельных БД**, миграций, доказательства концепции DR.

```bash
# Сжатый дамп одной БД (учётки из env или --uri)
mongodump --uri="mongodb://user:pass@host1:27017,host2:27017/?replicaSet=rs0&authSource=admin" \
  --db=shop --gzip --archive=/backup/shop_$(date +%Y%m%d).gz

# Восстановление в целевой кластер (осторожно с перезаписью)
mongorestore --uri="mongodb://..." --gzip --archive=/backup/shop_20250315.gz --nsInclude='shop.*'
```

**Production:** запускайте с **вторичного** при допустимой задержке или через **скрытый** член; контролируйте **IOPS**; храните архивы **вне** того же ЦОД с **encryption at rest**.

### Снимки файловой системы / блочного тома

- **Быстрее** для очень больших данных при поддерживаемом storage (EBS snapshot, LVM, SAN).
- Нужна **согласованность**: snapshot **тома с данными** при корректном завершении операций или **fs lock** / процедура vendor — уточняйте по гайдам MongoDB для вашего окружения.

### Point-in-time recovery (PITR) и oplog

- Непрерывный **oplog** на replica set позволяет накатывать изменения **после** момента логического бэкапа до времени сбоя.
- **Production:** для PITR нужны **регулярные** checkpoint’ы (`mongodump` / snapshot) **плюс** сохранение **oplog** (или managed backup с PITR).

---

## Частичный отказ кластера: как мыслить восстановление

| Сценарий | Ориентир действий |
|----------|-------------------|
| **Умер один secondary** | Заменить узел (новый диск/под), **добавить в RS** с тем же или новым именем по процедуре; кластер продолжал работу при кворуме |
| **Умер primary** | Обычно **автовыборы**; при деградации сети — восстановить связность или **вмешаться** по runbook (reconfig, priority) — только обученным |
| **Потерян majority** | Запись останавливается; **не** поднимайте «выдуманный» primary без понимания **split-brain** рисков — см. документацию **force reconfig** |
| **Повреждённые данные / drop коллекции** | Восстановление из **бэкапа** + **oplog** до нужной метки времени или **выборочный** `mongorestore` |

**Обязательно для production:**

1. **Runbook** с контактами, версией MongoDB, схемой RS/шардов.
2. **Регулярный тест restore** на staging (не только «дамп создаётся»).
3. **RPO/RTO** согласованы с бизнесом (что потеряем, за сколько восстановимся).
4. Мониторинг **репликационный лаг**, **opcounters**, **диск**, **выборы**.

---

## Production checklist (операционка)

| # | Практика |
|---|----------|
| 1 | Деплой через **IaC**; для K8s — **StatefulSet** + корректные **PVC** и лимиты |
| 2 | **TLS**, **auth**, **keyFile**/x509 между членами; секреты не в Git в открытом виде |
| 3 | Централизованные **логи** и **метрики** (в т.ч. exporter / cloud monitor) |
| 4 | Стратегия бэкапа: **logical** и/или **snapshot**, плюс понимание **PITR** |
| 5 | Письменные **DR**-упражнения и сценарий **partial failure** |

---

## Дополнительные материалы

- [Configuration File Options](https://www.mongodb.com/docs/manual/reference/configuration-options/)
- [Deploy Replica Set](https://www.mongodb.com/docs/manual/tutorial/deploy-replica-set/)
- [mongodump / mongorestore](https://www.mongodb.com/docs/database-tools/)
- [Backup Methods](https://www.mongodb.com/docs/manual/core/backups/)
- [MongoDB on Kubernetes](https://www.mongodb.com/docs/kubernetes-operator/stable/)
