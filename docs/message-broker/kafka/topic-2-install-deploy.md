# 2. Установка и deploy: bare metal, Docker и Kubernetes

---

Цель этой страницы — понять, как развернуть Kafka в production-сценариях и какие настройки особенно важны для отказоустойчивости и сетевой связности.

В production Kafka почти всегда — это:

- 3+ broker’а (минимум для осмысленного fault-tolerance),
- стабильный networking и корректные `advertised.*` адреса,
- persistent storage для данных и метаданных,
- управляемая архитектура отказоустойчивости (в т.ч. KRaft/KRaft node pools).

---

## Варианты деплоя

### Bare metal / VM

Подходит, когда вы хотите:

- максимальный контроль над сетью и дисками,
- предсказуемую latency,
- отдельное планирование отказоустойчивости (rack/zone).

Ключевые требования:

- выделенные диски/контроллеры под broker’а,
- надежная L3/L4 связность между broker’ами,
- корректная схема отказов: что произойдет при падении узла/сегмента сети.

### Docker (dev-only)

Docker удобен для обучения и экспериментов, но production почти никогда не упирается только в контейнеризацию:

- диски/IO ограничены,
- networking и DNS/DHCP особенности могут ломать `advertised.*` адреса для клиентов вне кластера,
- сложно воспроизвести реальные failure modes.

Best practice:

- используйте Docker для сценариев “проверить поведение”, но хранение и сетевую архитектуру тестируйте в environment, максимально похожем на production.

### Kubernetes

На Kubernetes обычно выбирают Strimzi (оператор), потому что он:

- стандартизирует развертывание,
- управляет Stateful workloads через PVC,
- упрощает корректные listener’ы/advertised адреса,
- дает production-рецепты (например, rack awareness).

---

## Production-настройки, о которых чаще всего “спотыкаются”

### 1) listeners и advertised.listeners: клиенты должны видеть правильные адреса

Почти все “Kafka не работает” в кластерах начинаются с того, что клиент получает адреса broker’ов, которые недоступны из его сети.

Best practice:

- внутри кластера отдавайте внутренние адреса,
- наружу (из других сетей/кластеров) — наружные/публичные адреса,
- для внешних маршрутов используйте overrides advertised адресов по broker ID.

Ниже — фрагмент Strimzi-логики для override advertised адресов (пример из официальной документации):

```yaml
# Встраивается в spec.kafka.listeners.configuration
configuration:
  # Подставляет {nodeId} в advertisedHostTemplate
  advertisedHostTemplate: example.hostname.{nodeId}
```

Комментарий:

- `advertisedHostTemplate` генерирует host для каждого broker’а, чтобы клиент гарантированно подключался по reachable имени.

### 2) rack awareness: распределяйте реплики по зонам/стойкам

Rack awareness снижает риск одновременного отказа нескольких реплик при проблемах в одной зоне.

Официальный pattern в Strimzi выглядит так:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
    rack:
      topologyKey: topology.kubernetes.io/zone
    # ...
```

Best practices:

- убедитесь, что Kubernetes nodes размечены лейблом зоны/стойки,
- проверьте schedule: broker’ы должны попадать в разные failure domains,
- если вы используете advanced перераспределение реплик — держите в уме KafkaRebalance/Cruise Control (для поддержания распределения).

### 3) Stateful workloads: PVC, storage класса и reclaim policy

В production:

- данные Kafka нельзя держать на ephemeral storage,
- качество диска (IOPS/latency) напрямую влияет на tail latency producer’ов/consumer’ов,
- reclaim policy и стратегия очистки при удалении кластера должны быть осознанными.

Best practice:

- применяйте PVC как часть “инфраструктурного контракта” (размер/класс диска/режим reclaim),
- выделяйте отдельные классы storage под разные типы нагрузок (если есть возможность).

---

## Kubernetes: Strimzi (KRaft) — каркас production-конфигурации

В Strimzi KRaft mode с node pools обычно включает:

- Kafka CR с аннотациями включения KRaft и node pools,
- KafkaNodePool для controller и broker ролей,
- и listener/rack/storage настройки в Kafka spec.

Ниже — пример “каркаса” (фрагменты), который показывает production-идею:

### Шаг 1: включить KRaft и node-pools на уровне Kafka CR

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-kafka-kraft-cluster
  annotations:
    strimzi.io/kraft: enabled
    strimzi.io/node-pools: enabled
spec:
  kafka:
    replicas: 0 # в node-pools-архитектуре обычно реплики задаются в KafkaNodePool
  # остальные секции: listeners, storage, config и т.д.
```

Комментарий:

- конкретные поля “storage/квоты/config” зависят от выбранного режима (JBOD/с типом хранения) и версии operator’а.

### Шаг 2: задать controller и broker roles через KafkaNodePool

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: controller
  labels:
    strimzi.io/cluster: my-kafka-kraft-cluster
spec:
  replicas: 3
  roles:
    - controller
  # storage, resources: задаются здесь
```

Best practice:

- для production обычно выделяют отдельные controller nodes (quorum) и отдельные broker nodes (data plane),
- controller quorum обычно держат из 3 нод (для доступности).

### Шаг 3: listeners и rack awareness в одном месте

Соберите listener overrides + rack section так, чтобы:

- клиенты (особенно извне кластера) видели корректные `advertised` адреса,
- реплики не оказывались в одной failure domain.

---

## Что важно для network topology и stateful workloads (short checklist)

- `advertised.listeners` адреса должны быть reachable из сети, где работают producer/consumer,
- node labels для rack awareness должны существовать и соответствовать `topologyKey`,
- PVC должны обеспечивать нужный IO профиль,
- количество broker’ов и размер кворума controllers соответствует требуемой устойчивости,
- тестируйте отказ: остановите один broker и проверьте поведение producer/consumer.

---

## Практика (быстрый старт в production-мышлении)

1) Определите failure domain’ы (zone/rack) и разметьте nodes.

```bash
kubectl label nodes <node-name> topology.kubernetes.io/zone=zone-a
```

2) Подготовьте Strimzi Kafka CR с KRaft annotations.
3) Убедитесь, что listeners/advertised адреса соответствуют вашему сетевому доступу.
4) Примените манифесты и проверьте:

- что broker’ы поднялись,
- что clients подключаются по advertised адресам,
- что rack awareness отражается в ожидаемом распределении реплик.

