# 4. Удалённые репозитории и синхронизация

**Цель темы:** уверенно работать с удалённым репозиторием (remote): добавлять его, настраивать upstream для веток, синхронизировать историю через fetch, push и pull и разрешать типичные ситуации при расхождении истории.

---

## Определения терминов

### Remote (удалённый репозиторий)

**Remote** — именованная ссылка на другой репозиторий (обычно на сервере: GitHub, GitLab, корпоративный сервер). По умолчанию после `git clone` создаётся remote с именем **origin**, указывающий на клонированный URL. С ним вы обмениваетесь коммитами через `git fetch`, `git push`, `git pull`.

### Remote-tracking ветка

**Remote-tracking branch** — локальная ссылка на состояние ветки на remote в момент последнего `git fetch` или `git pull`. Имена вида `origin/main`, `origin/feature/login`. Вы не коммитите в них напрямую; они обновляются только при fetch/pull. Ваша локальная ветка может «отслеживать» такую ветку (см. upstream).

### Upstream (восходящая ветка)

**Upstream** — связь локальной ветки с веткой на remote (чаще всего с remote-tracking веткой, например `origin/main`). Задав upstream, вы можете вызывать `git push` и `git pull` без указания имени remote и ветки; Git знает, куда пушить и откуда подтягивать.

### Fetch

**Fetch** — загрузка новых коммитов и обновление ссылок на ветки с remote (например, `origin/main`) **без** слияния их в вашу текущую ветку. Рабочая директория и текущая ветка не меняются. После fetch вы можете вручную сделать merge или rebase.

### Pull

**Pull** — по сути **fetch + merge** (или fetch + rebase при `git pull --rebase`). Загружает изменения с remote и вливает их в текущую ветку. Удобно для быстрой синхронизации, но важно понимать, что под капотом делается merge (или rebase).

### Push

**Push** — отправка ваших коммитов в удалённый репозиторий. Обновляет ветку на remote и соответствующие remote-tracking ветки у вас локально (после того как сервер примет push).

---

## Добавление и настройка remote

### Посмотреть список remote

```bash
git remote -v
# origin  https://github.com/user/repo.git (fetch)
# origin  https://github.com/user/repo.git (push)
```

### Добавить remote

```bash
git remote add origin https://github.com/user/repo.git
```

Имя `origin` — соглашение по умолчанию; можно использовать другое (например, `upstream` для оригинального репозитория при работе с форком).

### Изменить URL remote

```bash
git remote set-url origin https://gitlab.com/team/repo.git
```

### Несколько remote (форк + оригинал)

```bash
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git merge upstream/main   # подтянуть изменения из оригинального репозитория
```

---

## Fetch: обновление ссылок без слияния

Загрузить все изменения с remote и обновить remote-tracking ветки (`origin/main`, `origin/feature/x` и т.д.):

```bash
git fetch origin
git fetch --all          # все настроенные remote
git fetch -p              # после fetch удалить ссылки на удалённые ветки (prune)
```

После fetch ваши локальные ветки не меняются; вы решаете, как влить изменения: merge или rebase (см. ниже).

!!! tip "Практика"

    Перед началом работы полезно выполнить `git fetch origin` (или `git pull`), чтобы видеть актуальное состояние remote и избежать лишних конфликтов при push.

---

## Push: отправка коммитов на remote

### Первая отправка ветки (с настройкой upstream)

```bash
git push -u origin main
# или для feature-ветки:
git push -u origin feature/login
```

Опция `-u` (или `--set-upstream`) запоминает связь текущей ветки с `origin/main` (или `origin/feature/login`). Дальше можно вызывать просто `git push` и `git pull` без указания remote и имени ветки.

### Последующие push (upstream уже задан)

```bash
git push
```

### Отправить ветку под другим именем на remote

```bash
git push origin local-branch:remote-branch
# пример: локальная feature/foo → на origin как feature/bar
git push origin feature/foo:feature/bar
```

### Удалить ветку на remote

```bash
git push origin --delete feature/old
```

---

## Настройка upstream для существующей ветки

Если ветка уже есть на remote и вы клонировали репозиторий или создали ветку локально и хотите связать её с `origin/...`:

```bash
git branch --set-upstream-to=origin/feature/login feature/login
# или короче (текущая ветка):
git branch -u origin/feature/login
```

После этого `git status` может показывать «ahead/behind» относительно remote, а `git push` и `git pull` будут использовать этот upstream.

---

## Pull: подтянуть и влить изменения

```bash
git pull origin main     # fetch + merge origin/main в текущую ветку
git pull                # если upstream задан — подтянуть оттуда
git pull --rebase       # подтянуть и влить через rebase вместо merge
```

!!! warning "Внимание"

    `git pull` без аргументов использует upstream. Если upstream не задан, Git может выдать ошибку или спросить, откуда тянуть. Задайте upstream через `-u` при первом push или через `git branch --set-upstream-to`.

---

## Rejected push: расхождение истории

Если на remote появились новые коммиты, которых у вас нет, push будет отклонён:

```
! [rejected] main -> main (non-fast-forward)
error: failed to push some refs to '...'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart.
```

Нужно сначала подтянуть изменения с remote и только потом пушить.

### Вариант 1: pull (fetch + merge), затем push

```bash
git pull origin main   # подтянуть и смержить
# разрешить конфликты при необходимости
git push origin main
```

### Вариант 2: pull --rebase, затем push

```bash
git pull --rebase origin main
git push origin main
```

Ваши локальные коммиты «переедут» поверх актуального `origin/main`; история останется линейной.

!!! warning "Force push"

    `git push --force` перезаписывает ветку на remote. Используйте только в личных или согласованных сценариях (например, после локального rebase своей feature-ветки). **Не делайте force push в общие ветки** (main, develop) — это ломает историю у всех.

---

## Паттерны использования

| Паттерн | Описание |
|--------|----------|
| **Задавать upstream при первом push** | Всегда использовать `git push -u origin <ветка>`, чтобы потом вызывать `git push` и `git pull` без аргументов. |
| **Перед работой — fetch/pull** | Регулярно подтягивать изменения с origin, чтобы не накапливать расхождения и конфликты. |
| **Pull с rebase при личной ветке** | `git pull --rebase` даёт линейную историю без лишних merge-коммитов. |
| **Проверять статус перед push** | `git status` покажет «ahead/behind»; при «behind» сначала pull, потом push. |

---

## Антипаттерны

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Force push в общие ветки** | Стирает коммиты на remote, ломает клоны и CI у всей команды. | В общих ветках только обычный push после merge/pull. |
| **Игнорировать rejected push** | Повторный push без подтягивания не сработает; нужен осознанный pull/rebase. | Выполнить `git pull` (или `git pull --rebase`), разрешить конфликты, затем push. |
| **Не настраивать upstream** | Приходится каждый раз указывать `origin` и имя ветки; легко ошибиться. | Использовать `-u` при первом push или `git branch --set-upstream-to`. |
| **Долго не подтягивать с remote** | Большое расхождение ведёт к тяжёлым конфликтам при следующем pull/merge. | Регулярно делать fetch/pull, особенно перед началом новой задачи. |

---

## Примеры из production

### CI/CD и remote

Пайплайн клонирует репозиторий (`git clone` или `git fetch`), собирает артефакт из указанной ветки или коммита и иногда пушит теги или служебные коммиты. Понимание remote и push помогает настраивать права и триггеры (например, сборка только при push в `main` или в `release/*`).

### Защищённые ветки

На GitHub/GitLab для `main` включают защиту: запрет прямого push, обязательный merge через pull request, прохождение CI. В такой модели вы всегда пушите в свою ветку, а в main вливаете через merge request; расхождения с main вы решаете у себя через pull (rebase или merge) из origin/main.

### Синхронизация через pull request

Фича разрабатывается в ветке `feature/x`; вы пушите в `origin/feature/x`. Перед созданием pull request подтягиваете актуальный `main`: `git fetch origin && git merge origin/main` (или rebase). Так MR содержит актуальную разницу и меньше конфликтов при слиянии.

---

## Дополнительные материалы

- [Pro Git — глава 2.5 «Работа с удалёнными репозиториями»](https://git-scm.com/book/ru/v2/%D0%9E%D1%81%D0%BD%D0%BE%D0%B2%D1%8B-Git-%D0%A0%D0%B0%D0%B1%D0%BE%D1%82%D0%B0-%D1%81-%D1%83%D0%B4%D0%B0%D0%BB%D1%91%D0%BD%D0%BD%D1%8B%D0%BC%D0%B8-%D1%80%D0%B5%D0%BF%D0%BE%D0%B7%D0%B8%D1%82%D0%BE%D1%80%D0%B8%D1%8F%D0%BC%D0%B8)
- [git remote — документация](https://git-scm.com/docs/git-remote)
- [git push — документация](https://git-scm.com/docs/git-push)
- [git pull — документация](https://git-scm.com/docs/git-pull)
- [git fetch — документация](https://git-scm.com/docs/git-fetch)
