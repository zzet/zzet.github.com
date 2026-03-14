---
title: "git fetch: загрузка обновлений из удалённого репозитория"
description: "Полное объяснение git fetch. Чем отличается от git pull, как использовать --prune, работа с удалёнными ветками. Примеры команд."
date: 2026-01-11
lastmod: 2026-01-11
draft: false
slug: "git-fetch"
keywords: ["что такое fetch в git", "git fetch origin что это", "git fetch prune", "git fetch vs git pull"]
tags: ["git", "fetch", "intermediate"]
categories: ["git"]
---

Многие разработчики используют только `git pull` для обновления репозитория. Но `git fetch` — более безопасная альтернатива, которая даёт контроль над процессом синхронизации. Понимание разницы между ними меняет подход к работе с удалёнными репозиториями.

## Что делает git fetch

`git fetch` загружает изменения из удалённого репозитория — новые коммиты, ветки, теги — но **не меняет** вашу рабочую директорию и текущую ветку. Это безопасная операция только для чтения.

После fetch загруженные данные доступны через `origin/branch-name` — локальные копии удалённых веток. Ваша рабочая ветка остаётся нетронутой.

```bash
# Загрузить обновления из origin
git fetch origin

# После fetch: origin/main обновлён, но ваша main — нет
git log --oneline origin/main -5  # новые коммиты
git log --oneline main -5         # ваша ветка без изменений
```

## Git fetch vs git pull

`git pull` — это `git fetch` + `git merge` (или `git rebase`). Он скачивает и сразу сливает изменения.

```bash
# git pull — автоматически сливает
git pull origin main
# Эквивалентно:
git fetch origin main
git merge origin/main
```

Когда использовать `git fetch`:
- Хотите посмотреть что изменилось на сервере перед слиянием
- Боитесь неожиданных конфликтов при автоматическом merge
- Нужно обновить несколько веток, но слить только одну
- Работаете с большим репозиторием и хотите контролировать процесс

Когда `git pull` удобнее:
- Быстрое обновление в известном состоянии
- Нет незакоммиченных изменений
- Доверяете автоматическому слиянию

## Синтаксис git fetch

```bash
# Загрузить из origin (по умолчанию)
git fetch

# Загрузить из конкретного remote
git fetch origin

# Загрузить конкретную ветку
git fetch origin main

# Загрузить из всех remote
git fetch --all

# Загрузить и удалить устаревшие ссылки
git fetch --prune origin
git fetch -p origin
```

## Локальные копии удалённых веток (tracking branches)

После `git fetch` обновляются локальные копии удалённых веток. Они называются `origin/<branch>`:

```bash
# Посмотреть все ветки, включая удалённые
git branch -a

# Вывод:
# * main           ← ваша локальная ветка
#   feature/auth   ← локальная feature ветка
#   remotes/origin/main      ← копия удалённой main
#   remotes/origin/develop   ← копия удалённой develop
#   remotes/origin/feature/auth

# Посмотреть только удалённые ветки
git branch -r
```

`origin/main` — это не просто ссылка, а реальная локальная копия состояния удалённой ветки на момент последнего fetch.

## Работа с ветками после fetch

После fetch можно изучить новые изменения перед слиянием:

```bash
# Загрузить обновления
git fetch origin

# Посмотреть что изменилось на сервере
git log HEAD..origin/main --oneline

# Посмотреть diff между вашей main и удалённой
git diff main origin/main

# Слить если всё устраивает
git merge origin/main
# или
git rebase origin/main

# Создать локальную ветку от удалённой
git checkout -b feature origin/feature/new-api
# или
git switch -c feature origin/feature/new-api
```

## git fetch --prune: удаление устаревших ссылок

Когда удалённая ветка удаляется на сервере (например, после merge pull request), её локальная копия `origin/branch` остаётся у вас. Флаг `--prune` удаляет их:

```bash
# Обновить и удалить устаревшие локальные копии удалённых веток
git fetch --prune origin
git fetch -p origin

# Обновить все remote и удалить устаревшие
git fetch --all --prune

# Настроить автоматический prune при каждом fetch
git config --global fetch.prune true
```

После `--prune` в выводе появятся строки вида:
```
 - [deleted]         (none)     -> origin/feature/old-feature
```

## Fetch для fork workflow

При работе с форками нужно синхронизироваться с оригинальным репозиторием:

```bash
# Добавить оригинальный репо как upstream
git remote add upstream https://github.com/original/repo.git

# Загрузить обновления из upstream
git fetch upstream

# Посмотреть новые коммиты
git log HEAD..upstream/main --oneline

# Слить в вашу main
git checkout main
git merge upstream/main
# или через rebase
git rebase upstream/main
```

## Практические примеры

```bash
# Базовый fetch
git fetch origin

# Загрузить все remote
git fetch --all

# Загрузить и очистить устаревшие ветки
git fetch -p

# Проверить что появилось после fetch
git log origin/main --oneline -10

# Создать локальную ветку от загруженной
git checkout -b new-feature origin/feature/new-feature

# Сравнить с удалённой веткой
git diff origin/main

# Обновить без слияния (fetch + ручной merge позже)
git fetch origin main
git diff HEAD origin/main  # посмотреть различия
git merge origin/main       # слить когда готовы

# Загрузить теги
git fetch origin --tags

# Fetch с подробным выводом
git fetch -v origin

# Загрузить только определённую ветку
git fetch origin develop:local-develop
```

## Часто задаваемые вопросы

**Зачем нужен `git fetch`, если есть `git pull`?** `git fetch` даёт контроль: сначала видите что изменилось, потом решаете сливать или нет. `git pull` делает это автоматически. Fetch безопаснее при работе с активными ветками или когда есть незакоммиченные изменения.

**Что такое origin/master и чем это отличается от master?** `master` — ваша локальная ветка. `origin/master` — локальная копия состояния ветки `master` на сервере `origin` на момент последнего fetch. Они могут расходиться, если на сервере появились новые коммиты.

**Что означает флаг `--prune` в `git fetch --prune`?** Удаляет локальные копии удалённых веток (`origin/feature/*`), которые были удалены на сервере. Помогает поддерживать чистоту в `git branch -r`.

**Как создать локальную ветку на основе удалённой после fetch?** `git checkout -b local-name origin/remote-branch` или `git switch -c local-name origin/remote-branch`.

**Безопасен ли `git fetch`?** Да, абсолютно. Fetch только скачивает данные и обновляет локальные копии удалённых веток. Ваши файлы и текущая ветка не изменяются.

## Заключение

`git fetch` — инструмент для осознанной синхронизации. Сначала загрузите обновления, посмотрите что изменилось, затем решите как сливать. Это профессиональный подход к работе с удалёнными репозиториями. Для автоматического слияния — [git pull]({{< relref "git-pull" >}}). Для просмотра удалённых репозиториев — [git remote -v]({{< relref "git-remote-v" >}}).
