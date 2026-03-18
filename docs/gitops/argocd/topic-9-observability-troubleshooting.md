# 9. Наблюдаемость и troubleshooting

---
Middle+ обязан уметь дебажить: почему Argo CD показывает `OutOfSync`, почему health “красный”, или почему sync не применил изменения. В этой теме учимся расследовать проблемы через:

- логи Argo CD,
- события Kubernetes,
- diff view (сравнение desired/actual),
- и восстанавливать сломанные deploy’ы по воспроизводимой процедуре.

---

## Термины и сущности

| Термин | Определение |
|--------|-------------|
| **OutOfSync** | Argo CD обнаружил расхождение между desired state (Git) и actual state (кластер). |
| **Diff view** | UI/инструмент, который показывает различия между rendered manifests из Git и тем, что есть в кластере. |
| **Health status** | Оценка “здоровья” ресурсов/приложения (например, Healthy/Progressing/Degraded). |
| **Kubernetes Events** | Системные события кластера: scheduling failures, image pull errors, probe failures и т.д. |
| **Application sync** | Применение изменений из Git в кластер Argo CD. |
| **Reconciliation** | Процесс контроля желаемого состояния Argo CD: он регулярно сверяет diff и при необходимости инициирует sync. |

---

## Где смотреть в первую очередь (порядок действий)

Production-логика расследования:

1) понять, есть ли drift (`OutOfSync`)
2) посмотреть diff, чтобы понять “что именно должно измениться”
3) проверить health и конкретные ресурсы, которые “сломались”
4) проверить Kubernetes Events по проблемному ресурсу
5) если нужно — посмотреть логи Argo CD компонентов

---

## Логи Argo CD: какие компоненты важны

Обычно в Argo CD есть поды:

- `argocd-server` (API/UI, user actions)
- `argocd-repo-server` (Git fetch + render helm/kustomize)
- `argocd-application-controller` (reconcile/sync)

Команды:

```bash
# Логи приложения: сервер
kubectl -n argocd logs deploy/argocd-server --tail=200

# Логи render’а и Git fetch
kubectl -n argocd logs deploy/argocd-repo-server --tail=200

# Логи контроллера (sync/reconcile)
kubectl -n argocd logs deploy/argocd-application-controller --tail=200
```

Production best practice:

- храните ограничение на retention логов (иначе расследования будут “выедать” storage),
- включайте достаточно verbose логирования только на период расследования (а не постоянно).

---

## Diff view: что вы должны увидеть

Diff показывает различия between:

- **rendered manifests из Git**,
- **реальные объекты** в кластере.

Проверка CLI:

```bash
argocd app get orders-api

# Показать diff для ресурсов (идея)
argocd app diff orders-api
```

Как читать:

- если diff большой и “странный” — вероятна ошибка values/overlay (не тот environment),
- если diff маленький — чаще причина в конкретном ресурсе (immutable field, CRD readiness, RBAC запреты).

---

## Kubernetes Events: что реально “сломало” ресурс

Часто проблема не в Argo CD, а в том, что Kubernetes не смог создать/запустить ресурс:

- ImagePullBackOff,
- probe failures,
- scheduling/taints/insufficient resources,
- admission webhook rejects.

Команды:

```bash
# События в namespace приложения
kubectl -n production get events --sort-by='.lastTimestamp'

# Проблемный pod (если известен)
kubectl -n production describe pod <pod-name>
```

Best practice:

- смотрите Events именно для “сломанных” ресурсов из Argo CD (а не по всему namespace).

---

## Практика: сломать Deployment и восстановить

### 1) Создать controlled drift

Например, уменьшим replicas вручную:

```bash
kubectl -n production scale deployment/orders-api --replicas=0
```

Ожидание:

- Argo CD через reconcile заметит расхождение и покажет `OutOfSync`.

### 2) Найти причину через diff/health

```bash
argocd app get orders-api
argocd app diff orders-api
```

### 3) Восстановить (sync обратно)

```bash
argocd app sync orders-api
argocd app wait orders-api --health
```

Production best practice:

- восстановление должно быть “Git-first”: sync возвращает desired state из репозитория.
- если sync не помогает — значит проблема в применении (RBAC, webhook, CRD readiness, immutable fields).

---

## Best practices: как сделать troubleshooting предсказуемым

| Практика | Зачем |
|----------|-------|
| Всегда проверять diff перед action | Уменьшает время на “догадки”. |
| Сопоставлять health/sync status с конкретными ресурсами | “Красный health” без ресурсов — пустая трата времени. |
| Смотреть Kubernetes Events по проблемным ресурсам | Это часто быстрее, чем читать логи Argo CD целиком. |
| Логи компонентов Argo CD — только по необходимости | Не захламляйте хранение логов. |
| Ограничивать self-heal/prune в production | Иначе расследование станет “борьбой с автоматикой”. |

---

## Дополнительные материалы

- [Argo CD — Troubleshooting](https://argo-cd.readthedocs.io/en/stable/troubleshooting/)
- [Kubernetes — Events](https://kubernetes.io)

