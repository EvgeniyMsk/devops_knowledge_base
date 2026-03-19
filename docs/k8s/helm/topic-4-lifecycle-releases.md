# 4. Lifecycle и управление релизами

---
Релиз в Helm — это **именованная установка** chart’а с **историей ревизий** (`revision`). **Upgrade** добавляет новую ревизию и применяет манифесты в кластере; **rollback** откатывает к выбранной ревизии. Ниже — команды **`list` / `history` / `rollback`**, как устроен **upgrade**, что происходит при **откате** и зачем **`--atomic`** в production.

---

## Где хранится состояние (Helm v3)

Метаданные релиза (манифест, values, chart, статус) сохраняются в **Secrets** (по умолчанию) или **ConfigMaps** в namespace релиза, с лейблом `owner=helm`. Это и есть **история** для `helm history` и **rollback**.

Best practice: ограничивать **число ревизий** (`--history-max` при upgrade/install), чтобы не раздувать etcd; типично **5–20** зависит от политики.

---

## Основные команды

### Список релизов

```bash
helm list -A
helm list -n production
```

### История ревизий

```bash
helm history my-app -n production
```

Колонки: **REVISION**, **STATUS** (`deployed`, `failed`, `superseded`…), **CHART**, короткое описание.

### Откат

```bash
helm rollback my-app 7 -n production
```

Восстанавливается **манифест ревизии 7**; Helm создаёт **новую** ревизию (например 12), которая по содержанию совпадает с 7.

```bash
helm rollback my-app 7 -n production --wait --timeout 5m
```

---

## Как работает `helm upgrade`

1. Загружается chart (локально или из репозитория).
2. Сливаются **values** (см. раздел про приоритет `-f` / `--set`).
3. Рендерятся шаблоны → итоговые YAML.
4. Сравнение с **живым состоянием** кластера (**three-way merge** с последней успешной ревизией): Helm старается **сохранить** поля, добавленные вручную (например через `kubectl patch`), где это применимо — поведение зависит от ресурса и аннотаций.
5. Применение к API сервера (часто **server-side apply** в современных версиях).
6. Запись новой **ревизии** в историю релиза.

Комментарий: «ручные» правки в кластере могут **конфликтовать** со следующим upgrade — в production предпочтительно **GitOps** или чёткая дисциплина (не править контролируемые поля вручную).

---

## Что происходит при rollback

- Берётся **снимок манифестов** указанной ревизии (даже если промежуточные upgrade были «частичными» — ориентируйтесь на **status** в `helm history`).
- Накатывается откат как **новый** revision в истории.
- **Hooks** (`pre-rollback`, `post-rollback`) выполняются, если они есть в chart’е.

Best practice: перед rollback на инциденте сохранить **`helm get manifest`** и **`helm get values`** текущей (битой) ревизии в тикет.

---

## Атомарный деплой: `--atomic`

| Поведение | Смысл |
|-----------|--------|
| **`--atomic`** | Если **установка/upgrade** не завершается успехом в рамках **`--timeout`**, Helm **откатывается** к последней рабочей ревизии |
| **`--wait`** | Ждать готовности Pods / Job hook’ов (зависит от ресурсов и hook’ов) |
| **`--timeout`** | Максимум ожидания (например `5m`, `10m`) |

```bash
helm upgrade --install my-app ./my-app -n production \
  -f values-prod.yaml \
  --atomic \
  --wait \
  --timeout 10m
```

Production: для критичных релизов комбинируйте **`--atomic` + `--wait` + разумный `--timeout`**; учитывайте **long-running** hook’и и **PDB**.

Смежный флаг: **`--cleanup-on-fail`** — подчистка ресурсов при неудачном install/upgrade (см. документацию к вашей версии Helm).

---

## Диагностика релиза

```bash
helm status my-app -n production
helm get values my-app -n production
helm get manifest my-app -n production --revision 11
helm get notes my-app -n production
```

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | Ограничить `--history-max`, мониторить число helm release secrets |
| 2 | CI хранит **номер chart version** и **revision** деплоя в артефакте/тикете |
| 3 | Критичные пути — **`--atomic`** и явный **`--timeout`** |
| 4 | Runbook: **rollback** + кого звать, если rollback не помог (данные, миграции) |
| 5 | Не править через `kubectl edit` поля, которыми **владеет** Helm, без процесса |

---

## Дополнительные материалы

- [Helm upgrade](https://helm.sh/docs/helm/helm_upgrade/)
- [Helm rollback](https://helm.sh/docs/helm/helm_rollback/)
- [Install — atomic, wait, timeout](https://helm.sh/docs/intro/using_helm/#helm-install-installing-a-chart)

