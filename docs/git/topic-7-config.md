# 7. Конфигурация и практики

**Цель темы:** настраивать Git под себя и под политики команды: конфигурация (user, alias, типовые опции), хуки для проверок перед коммитом и push, подписанные коммиты (GPG) и типовые практики работы с несколькими remote.

---

## Определения терминов

### Уровни конфигурации Git

**Системный** (`--system`) — для всех пользователей на машине, обычно в `/etc/gitconfig`.  
**Глобальный** (`--global`) — для текущего пользователя, хранится в `~/.gitconfig` (или `~/.config/git/config`).  
**Локальный** (репозиторий) — только для данного репозитория, файл `.git/config`.  

Локальные настройки переопределяют глобальные, глобальные — системные.

### Хук (hook)

**Хук** — скрипт, который Git запускает при определённом событии (commit, push, merge и т.д.). Лежат в `.git/hooks/`; имена фиксированы (`pre-commit`, `commit-msg`, `pre-push` и др.). Если скрипт завершается с ненулевым кодом, операция прерывается. Используются для линтинга, проверки формата сообщений, запуска тестов перед push.

### Подписанный коммит (signed commit)

**Подписанный коммит** — коммит, к которому привязана криптографическая подпись (обычно GPG). Подтверждает, что автор владеет соответствующим ключом. На GitHub/GitLab такие коммиты помечаются как «Verified» и повышают доверие к истории (compliance, защита от подделки авторства).

### Signed-off-by

**Signed-off-by** — строка в сообщении коммита (часто добавляется хуком или вручную), указывающая подписавшего изменения. Используется в проектах с DCO (Developer Certificate of Origin) и в некоторых workflow код-ревью.

---

## Конфигурация: git config

### Имя и email (обязательно для коммитов)

Без них Git откажется создавать коммит или предупредит:

```bash
git config --global user.name "Иван Иванов"
git config --global user.email "ivan@example.com"
```

Для одного репозитория (например, рабочий проект с корпоративной почтой):

```bash
git config user.email "ivan@company.com"
```

### Просмотр и сброс

```bash
git config --list              # все настройки (все уровни)
git config user.name           # значение одной опции
git config --global --unset user.email   # удалить опцию
```

### Полезные алиасы

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.lg "log --oneline -20"
git config --global alias.last "log -1 HEAD"
```

После этого: `git st`, `git co main`, `git lg`, `git last`.

### Типовые опции

```bash
# Редактор по умолчанию (для сообщений коммита, rebase -i)
git config --global core.editor "code --wait"

# Ветка по умолчанию при git init
git config --global init.defaultBranch main

# pull по умолчанию через rebase вместо merge
git config --global pull.rebase true

# Цветной вывод
git config --global color.ui auto
```

!!! tip "Практика"

    В команде договоритесь об общих правилах: формат сообщений коммитов (например, Conventional Commits), имя основной ветки (`main`/`master`), использование rebase при pull. Часть можно enforced через хуки и CI.

---

## Хуки (hooks)

### Расположение

Скрипты лежат в `.git/hooks/`. Имя файла = имя хука. Примеры: `pre-commit`, `commit-msg`, `pre-push`, `post-merge`. Исполняемый бит должен быть установлен: `chmod +x .git/hooks/pre-commit`.

### pre-commit

Запускается перед созданием коммита. Если скрипт возвращает ненулевой код, коммит отменяется.

Пример: проверить, что не коммитятся отладочные выводы и секреты:

```bash
#!/bin/sh
# .git/hooks/pre-commit
if git diff --cached --name-only | xargs grep -l "console.log\|TODO:FIXME\|password\s*=" 2>/dev/null; then
  echo "Commit contains forbidden patterns. Remove them."
  exit 1
fi
exit 0
```

Часто в pre-commit запускают линтер и форматтер (eslint, black, gofmt), чтобы не допустить «грязный» код в историю.

### commit-msg

Получает путь к временному файлу с сообщением коммита. Можно проверять формат (например, «тип: описание» или длина первой строки).

```bash
#!/bin/sh
# .git/hooks/commit-msg
msg=$(cat "$1")
if ! echo "$msg" | head -1 | grep -qE '^(feat|fix|docs|style|refactor|test|chore): .{10,}'; then
  echo "Commit message must start with type: feat, fix, docs, ... and be at least 10 chars."
  exit 1
fi
exit 0
```

### pre-push

Запускается перед отправкой на remote. Удобно запускать тяжёлые тесты или проверки стиля только перед push, а не при каждом коммите.

```bash
#!/bin/sh
# .git/hooks/pre-push
npm run test || exit 1
exit 0
```

!!! warning "Внимание"

    Хуки в `.git/hooks/` не клонируются и не пушатся — они локальные. Чтобы распространить их по команде, используют шаблон директории (`core.hooksPath`), общий скрипт в репозитории (например, `scripts/setup-hooks.sh`) или фреймворк pre-commit (см. ниже).

---

## Pre-commit framework

[pre-commit](https://pre-commit.com/) — фреймворк для управления хуками. Конфигурация в репозитории (файл `.pre-commit-config.yaml`), хуки ставятся командой `pre-commit install`. Так все разработчики получают одинаковый набор проверок.

Пример конфигурации:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-added-large-files
      - id: detect-private-key
  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
        language_version: python3
```

Установка: `pip install pre-commit`, затем в корне репо `pre-commit install`. Хуки будут запускаться при каждом коммите.

---

## Подписанные коммиты (GPG)

### Зачем

Подпись подтверждает авторство и целостность коммита. На GitHub/GitLab коммиты помечаются как «Verified»; в защищённых репозиториях можно требовать только подписанные коммиты.

### Настройка

1. Создать GPG-ключ (если нет): `gpg --full-generate-key`.
2. Указать ключ в Git:

```bash
git config --global user.signingkey <key-id>
# key-id — последние 8 символов от gpg --list-secret-keys --keyid-format LONG
```

3. Подписывать коммиты по умолчанию:

```bash
git config --global commit.gpgsign true
```

4. Добавить публичный ключ в аккаунт GitHub/GitLab (Settings → SSH and GPG keys).

### Разовое подписание коммита

```bash
git commit -S -m "Signed commit message"
```

При включённом `commit.gpgsign` флаг `-S` не обязателен.

!!! example "Production"

    В репозиториях с высокими требованиями к compliance включают «Require signed commits» в настройках ветки. Тогда push неподписанных коммитов будет отклонён.

---

## Прочие практики

### Отмена merge до коммита

Если начали merge, возникли конфликты и решили отменить:

```bash
git merge --abort
```

### Несколько remote: fork + upstream

При работе с форком добавляют remote на оригинальный репозиторий и подтягивают изменения оттуда:

```bash
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git merge upstream/main
# или
git rebase upstream/main
```

Обновление форка: регулярно `git fetch upstream && git merge upstream/main` (или rebase) в своей ветке.

---

## Паттерны использования

| Паттерн | Описание |
|--------|----------|
| **Единые стандарты в команде** | Договориться о формате сообщений, имени основной ветки, использовании rebase; закрепить в хуках и CI. |
| **Хуки для быстрых проверок** | Линтер, форматтер, проверка сообщения в pre-commit; тяжёлые тесты — в pre-push или в CI. |
| **Pre-commit / общий setup-hooks** | Чтобы хуки были у всех, хранить конфиг в репо и скрипт установки или использовать pre-commit. |
| **Подписанные коммиты в защищённых репо** | Включить подпись по умолчанию и требовать Verified в настройках ветки. |

---

## Антипаттерны

| Антипаттерн | Почему плохо | Что делать |
|-------------|--------------|------------|
| **Обходить хуки через --no-verify** | Постоянное использование `git commit --no-verify` ломает политику качества. | Исправлять причину срабатывания хука или временно отключить конкретную проверку по договорённости. |
| **Разные форматы коммитов в команде** | Хаотичная история, сложнее автоматизация (changelog, релизы). | Ввести единый формат и проверять в commit-msg или CI. |
| **Хуки только у части команды** | У кого-то проверки есть, у кого-то нет — неравные условия. | Распространять хуки через pre-commit или скрипт в репо и CI как страховку. |
| **Игнорировать предупреждения о user.name/email** | Коммиты с пустым или неправильным автором портят историю и статистику. | Настроить глобально или локально для репо. |

---

## Примеры из production

### Обязательные проверки в репозитории

В репо лежит `.pre-commit-config.yaml` и в README/onboarding указано: установить pre-commit и выполнить `pre-commit install`. В CI те же проверки дублируются (lint, format), чтобы поймать коммиты, сделанные с `--no-verify` или без установленных хуков.

### Signed commits в защищённых ветках

Для `main` включена опция «Require signed commits». Разработчики настраивают GPG и `commit.gpgsign true`. Merge request с неподписанными коммитами не принимается или показывается предупреждение.

### Разный email для работы и личных проектов

Глобально — личный email; в рабочем репо переопределение: `git config user.email "ivan@company.com"`. Так коммиты в корпоративном репо идут с рабочей почтой, в личных — с личной.

---

## Дополнительные материалы

- [Pro Git — глава 8.1 «Конфигурация Git»](https://git-scm.com/book/ru/v2/%D0%9E%D1%81%D0%BD%D0%BE%D0%B2%D1%8B-Git-%D0%9A%D0%BE%D0%BD%D1%84%D0%B8%D0%B3%D1%83%D1%80%D0%B0%D1%86%D0%B8%D1%8F-Git)
- [Pro Git — глава 8.3 «Git Hooks»](https://git-scm.com/book/ru/v2/%D0%9E%D1%81%D0%BD%D0%BE%D0%B2%D1%8B-Git-%D0%9F%D0%BE%D0%B4%D0%BF%D0%B8%D1%81%D1%8C-%D0%BA%D0%BE%D0%BC%D0%BC%D0%B8%D1%82%D0%BE%D0%B2)
- [git config — документация](https://git-scm.com/docs/git-config)
- [Git Hooks — документация](https://git-scm.com/docs/githooks)
- [Pre-commit](https://pre-commit.com/) — фреймворк для хуков
- [Signing commits — GitHub](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits)
- [Conventional Commits](https://www.conventionalcommits.org/) — формат сообщений коммитов
