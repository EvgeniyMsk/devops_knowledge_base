# 7. Безопасность

---
В **production** MongoDB не должна быть доступна «как сосед по VLAN без пароля»: нужны **аутентификация** (типично **SCRAM**), **авторизация по ролям**, **TLS** в пути данных и **сетевая изоляция**. Ниже — модель угроз в двух словах, практика в **mongosh** и минимальный чеклист.

---

## Аутентификация: SCRAM

**SCRAM** (**Salted Challenge Response Authentication Mechanism**) — современный механизм по умолчанию: пароль **не** уходит по сети в открытом виде, используется challenge-response.

| Практика | Комментарий |
|----------|-------------|
| **Включить `--auth`** (или `security.authorization: enabled`) | После создания первого администратора в admin |
| **Отдельные пользователи** на приложение, бэкап, мониторинг | Не делиться одной учёткой `root` |
| **Пароли** из секрет-хранилища | Ротация, длина, запрет в Git |

**Best practice:** при первом старте кластера используйте процедуру из документации «создать пользователя до включения auth» или локальное подключение по **localhost exception** там, где она ещё поддерживается вашей версией.

---

## Авторизация: встроенные и кастомные роли

Доступ задаётся **ролями** в **`admin`** или других БД. Встроенные роли (фрагмент модели):

| Роль (пример) | Назначение |
|----------------|------------|
| `read` / `readWrite` | Чтение / чтение+запись в **конкретной** БД |
| `dbAdmin` | Администрирование одной БД (индексы, статистика) |
| `clusterMonitor` | Метрики для мониторинга без изменения данных |
| `backup` / `restore` | Ограниченный набор для резервного копирования |
| `userAdmin` | Управление пользователями (осторожно) |

**Production:** **least privilege** — приложению только **`readWrite`** на нужную БД (или **`read`** для read-only сервисов); отдельный пользователь с **`clusterMonitor`** для экспортера.

### Создание пользователей и ролей

```javascript
// Под администратором в admin
use admin

// Приложение: только одна БД
db.createUser({
  user: "shop_app",
  pwd: "ИСПОЛЬЗУЙТЕ_СЕКРЕТ_ИЗ_VAULT",
  roles: [{ role: "readWrite", db: "shop" }],
})

// Мониторинг: только метрики (имя роли зависит от версии; уточняйте в документации)
db.createUser({
  user: "shop_monitor",
  pwd: "REDACTED",
  roles: [{ role: "clusterMonitor", db: "admin" }],
})

// Просмотр прав текущего пользователя
db.runCommand({ connectionStatus: 1 })
```

Кастомная роль (пример ограниченного набора привилегий):

```javascript
use admin
db.createRole({
  role: "shopSupportRead",
  privileges: [
    {
      resource: { db: "shop", collection: "" },
      actions: ["find", "listCollections"],
    },
  ],
  roles: [],
})

db.createUser({
  user: "support_ro",
  pwd: "REDACTED",
  roles: [{ role: "shopSupportRead", db: "admin" }],
})
```

---

## TLS (шифрование в пути)

- **TLS** между клиентами и `mongod`, между членами replica set (рекомендуется **x509** или общий подход с CA в enterprise).
- Сертификаты: валидные **SAN** для DNS-имён подов/сервисов в Kubernetes.

Фрагмент **`mongod.conf`** (идея):

```yaml
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongo.pem
    CAFile: /etc/ssl/ca.pem
```

**Best practice:** `allowInvalidCertificates` только в **лаборатории**; на бою — полноценная проверка цепочки.

---

## Сетевая изоляция

| Мера | Замечание |
|------|-----------|
| **Bind / firewall** | `mongod` слушает только доверенные интерфейсы; **SG**/ACL «только от приложений» |
| **Не** выставлять порт 27017 в интернет | Боты и brute-force |
| **Private** сети между узлами RS | Уменьшение поверхности атаки |
| **VPN / bastion** для админского доступа | Вместо прямого `mongosh` из публичной сети |

---

## Строка подключения с аутентификацией

```text
mongodb://shop_app:SECRET@mongo-0:27017,mongo-1:27017/shop?replicaSet=rs0&authSource=shop&tls=true
```

- **`authSource`** — БД, где заведён пользователь (часто совпадает с БД приложения или `admin`).
- Для **TLS** добавьте параметры, ожидаемые вашим драйвером (`tls`, `tlsCAFile`).

---

## Практика: включить auth и ограничить доступ

1. Поднять узлы в **закрытой** сети; добавить **TLS** по мере готовности CA.
2. Создать пользователей **`admin`** (роль `root` или раздельно `userAdmin` + опер. роли — по политике).
3. Создать **прикладных** пользователей с минимальными ролями.
4. Включить **`authorization: enabled`** и **перезапустить** (или rolling restart RS).
5. Убедиться, что анонимное подключение **запрещено**, приложения ходят с **отдельными** URI из секретов.

Проверка:

```bash
# Без учётных данных — ожидаем отказ при включённом auth
mongosh "mongodb://mongo:27017"

# С учёткой приложения — успех
mongosh "mongodb://shop_app:REDACTED@mongo:27017/shop?authSource=shop"
```

---

## Production checklist (безопасность)

| # | Практика |
|---|----------|
| 1 | **Auth + TLS** на всех контурах кроме изолированного dev |
| 2 | Роли по **least privilege**; аудит кто имеет **`readWrite`** / **admin** |
| 3 | Секреты в **Vault/K8s Secret**; не коммитить пароли |
| 4 | Регулярные **обновления** MongoDB (CVE); контроль **bindIp** и firewall |
| 5 | Логи аутентификации и подозрительная активность — в SIEM при наличии |

---

## Дополнительные материалы

- [Enable Authentication](https://www.mongodb.com/docs/manual/tutorial/enable-authentication/)
- [SCRAM](https://www.mongodb.com/docs/manual/core/security-scram/)
- [Role-Based Access Control](https://www.mongodb.com/docs/manual/core/authorization/)
- [TLS/SSL](https://www.mongodb.com/docs/manual/core/security-transport-encryption/)
