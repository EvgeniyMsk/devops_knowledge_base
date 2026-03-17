# Git
![Git[200]](./images/welcome.png)

Git — распределённая система управления версиями. Проект был создан Линусом Торвальдсом для управления разработкой ядра Linux, первая версия выпущена 7 апреля 2005 года; координатор — Дзюн Хамано.

## Разделы

- [**1. Необходимый базис Git**](topic-1-basis.md) — что такое Git, распределённая СКВ, репозиторий, рабочая директория, индекс, коммит, ветка, HEAD; структура `.git`; базовые команды `init`, `add`, `commit`, `status`, `log`; паттерны и антипаттерны; production.

- [**2. Работа с изменениями и индекс (staging)**](topic-2-staging.md) — индекс в деталях, `git add` / `git restore --staged`, просмотр различий (`git diff`, `git diff --staged`), `git commit --amend`, `.gitignore`; паттерны и антипаттерны; production.

- [**3. Ветки и слияние**](topic-3-branches.md) — создание и удаление веток, merge и rebase, fast-forward, разрешение конфликтов; паттерны и антипаттерны; production (стратегии ветвления, защита main).

- [**4. Удалённые репозитории и синхронизация**](topic-4-remote.md) — remote, remote-tracking ветки, fetch, push, pull, upstream; rejected push и его разрешение; паттерны и антипаттерны; production (CI/CD, защищённые ветки).

- [**5. Отмена, история и восстановление**](topic-5-undo.md) — reset (--soft, --mixed, --hard), revert, reflog; восстановление «потерянных» коммитов и веток; revert vs reset для общих веток; паттерны и антипаттерны; production.

- [**6. Продвинутые операции**](topic-6-advanced.md) — stash, cherry-pick, интерактивный rebase, bisect, blame; паттерны и антипаттерны; production (чистая история перед MR, поиск регрессии, аудит).

- [**7. Конфигурация и практики**](topic-7-config.md) — git config (уровни, user, alias), хуки (pre-commit, commit-msg, pre-push), pre-commit framework, подписанные коммиты (GPG), несколько remote; паттерны и антипаттерны; production.

- [**8. Git в CI/CD и автоматизации**](topic-8-cicd.md) — shallow clone, теги, sparse checkout, подмодули; переменные и сценарии в CI; безопасность (токены, секреты); паттерны и антипаттерны; production (деплой по тегу, версирование артефактов).
