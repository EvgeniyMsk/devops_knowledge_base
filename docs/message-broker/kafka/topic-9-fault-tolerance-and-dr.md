# 9. Отказоустойчивость и DR

---

В этой теме — как Kafka ведёт себя при отказах и как производственно тренировать recovery-процедуры.

Сценарии:

- broker down,
- network partition,
- disk full,
- и связанные эффекты: leader election, ISR, rebalance у consumer group’ов.

Инструменты: MirrorMaker 2 (межкластерная репликация для DR).

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **Leader** | Узел, который обслуживает запись/чтение для конкретной части данных. |
| **ISR (in-sync replicas)** | Реплики, которые синхронизированы с leader’ом и считаются “безопасными” для продолжения записи. |
| **Rebalance** | Перераспределение разделов данных между членами consumer group’а при изменениях членства или недоступности частей кластера. |
| **Unclean leader election** | Режим, при котором могут быть выбраны не-синхронизированные реплики (повышает доступность, но может ухудшить консистентность). |
| **DR (Disaster Recovery)** | Восстановление сервиса после крупной аварии с RTO/RPO, обычно с межкластерной репликацией. |

---

## Сценарии отказов: что ожидать и что проверять

### 1) Broker down

Что происходит:

- leader может переизбраться на доступную реплику,
- запись продолжится, если ISR и `min.insync.replicas` позволяют,
- consumer group’ы сделают rebalance и продолжат чтение с последнего committed прогресса (или с сохранённых offset’ов).

Production best practices:

- заранее закладывайте число broker’ов и replication factor под отказ “n узлов”,
- для критичных потоков используйте `acks=all` на producer’е и корректный `min.insync.replicas`,
- не включайте “нечистую” переизбрание leader’ов без бизнес-обоснования.

### Симуляция: остановить один broker (Kubernetes)

```bash
# 1) Найдите broker под’ы
kubectl -n kafka get pods -l strimzi.io/name=my-cluster-kafka -o name

# 2) Удалите (эмулируем падение)
kubectl -n kafka delete pod <broker-pod-name>

# 3) Наблюдайте: новые leaders/ISR и rebalance
kubectl -n kafka describe pod <consumer-pod-name>
```

Комментарий:

- в production вы обычно “убиваете” через контролируемые операции (cordon/drain), но для практики полезно воспроизвести отказ именно как потерю pod’а.

---

### 2) Network partition

Что происходит:

- часть broker’ов станет недоступной по сети,
- ISR может “усыхать” (репликации не синхронизируются),
- возможны ошибки producer’ов (timeouts/retries) и рост задержек у consumer’ов.

Production best practices:

- планируйте failure domains (rack/zone) и используйте rack awareness,
- настраивайте timeouts/retries так, чтобы recovery занимал предсказуемое время,
- учитывайте, что partition может быть “частичным” (не равномерным).

#### Симуляция: временно заблокировать доступ к порту broker’а

```bash
# Пример (идея): блокируем исходящие TCP на 9092 с pod’а consumer’а.
# Реальные команды зависят от вашего базового образа и наличия iptables.

kubectl -n kafka exec -it <consumer-pod-name> -- sh -c \
  "iptables -A OUTPUT -p tcp --dport 9092 -j DROP"

# Через N минут снимите правило
kubectl -n kafka exec -it <consumer-pod-name> -- sh -c \
  "iptables -D OUTPUT -p tcp --dport 9092 -j DROP"
```

---

### 3) Disk full

Что происходит:

- broker начинает ограничивать операции из‑за заполнения,
- grow latency и error rate,
- retention cleanup может отставать (особенно при compaction/cleaner нагрузке).

Production best practices:

- мониторьте disk usage и задавайте thresholds для раннего вмешательства,
- планируйте рост storage как часть capacity planning,
- тренируйте реакцию: увеличить объем, перестроить storage, проверить cleaner throughput.

#### Симуляция: заполнить тестовый mount (dev/prod-drill)

```bash
# Заполнить файл до лимита на mounted volume (примерная идея)
kubectl -n kafka exec -it <broker-pod-name> -- sh -c \
  "dd if=/dev/zero of=/var/lib/kafka/fill-test bs=1M status=progress"
```

Комментарий:

- выполняйте только на staging или в controlled drill, иначе можно сломать cluster-wide SLO.

---

## Leader election: как проверять recovery

Ключевая мысль: leader переизбирается автоматически, но ваша задача — убедиться, что:

- потребители продолжили читать,
- запись продолжается без длительных ошибок,
- ISR восстановился до ожидаемого состояния (если политика требует).

Практика проверки:

```bash
# Быстро оценить consumer progress
kafka-consumer-groups.sh \
  --bootstrap-server <bootstrap> \
  --describe \
  --group <group-name>
```

Рекомендации:

- сравнивайте до/после recovery метрики latency/error/lag,
- фиксируйте “как долго заняло восстановление” для RTO.

---

## DR между кластерами: MirrorMaker 2 (инструмент)

MirrorMaker 2 нужен, когда вы хотите:

- реплицировать данные между кластерами в разных failure domains (например, разные дата-центры),
- пережить крупную аварию: кластер A недоступен, но кластер B продолжает обслуживать потребителей.

Production best practices:

- определите RPO: как много данных можно потерять по факту недоставки,
- определите routing потребителей при DR (какой consumer group переключается на второй кластер),
- заранее протестируйте switch и проверьте совместимость схем/семантики.

Примечание по коду:

- конкретная настройка MirrorMaker 2 обычно включает фильтрацию потоков/групп; поля зависят от выбранного формата и конфигурационного файла вашего кластера.

---

## План drill: “сломать и восстановить” (шаблон)

1) Определите SLA восстановления: например, broker down за <N минут, network partition за <M минут.
2) Выполните сценарий (подвес, блок сети, заполнение диска в staging).
3) Соберите признаки:
   - ошибки producer’ов,
   - рост задержек,
   - изменение прогресса consumer group’ов,
   - восстановление доступности записи/чтения.
4) Зафиксируйте результат и внесите изменения в:
   - timeouts/retries,
   - replication/ISR параметры,
   - capacity (storage/IO).

Небольшая проверка прогресса:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server <bootstrap> \
  --describe \
  --group <group-name>
```

---

## Production checklist

| Что проверить | Зачем |
|--------------|-------|
| replication/ISR политики под ожидаемые отказа | запись продолжалась без потери SLO |
| producer retry/timeout поведение при недоступности | меньше “тишины” и падений |
| disk usage алерты и runbook | чтобы заполнение не стало “сюрпризом” |
| drill сценарии раз в период | recovery становится предсказуемым |
| межкластерный plan для DR и тест switch | чтобы RTO/RPO реально достигались |

