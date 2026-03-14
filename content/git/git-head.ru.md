---
title: "HEAD в Git: что это такое и как с ним работать"
description: "Что такое HEAD в Git. Detached HEAD, нотация HEAD~1 и HEAD^, просмотр и перемещение HEAD. Как вернуться из detached HEAD состояния."
date: 2026-01-13
lastmod: 2026-01-13
draft: false
slug: "git-head"
keywords: ["git head что это", "HEAD указатель git", "detached HEAD git", "что такое HEAD в git", "HEAD~1 git что это", "git HEAD указатель на коммит"]
tags: ["git", "intermediate"]
categories: ["git"]
---

HEAD — это специальный указатель в Git, который показывает «где вы находитесь сейчас» в репозитории. Обычно HEAD указывает на текущую ветку, а та — на последний коммит. Понимание HEAD важно для работы с историей, cherry-pick, rebase и восстановления коммитов.

## Что такое HEAD

HEAD — файл в `.git/HEAD`, который содержит ссылку на текущую ветку или напрямую на хеш коммита:

```bash
# Посмотреть содержимое HEAD
cat .git/HEAD
# ref: refs/heads/main
# (указывает на ветку main)

# При detached HEAD:
cat .git/HEAD
# a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
# (напрямую на коммит)
```

Обычный HEAD: `HEAD → main → коммит A`
Detached HEAD: `HEAD → коммит A` (ветку минует)

## Просмотр HEAD

```bash
# Хеш коммита на который указывает HEAD
git rev-parse HEAD
# a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0

# Сокращённый хеш
git rev-parse --short HEAD
# a1b2c3d

# Имя ветки HEAD (или "(HEAD detached at ...)")
git symbolic-ref --short HEAD
# main

# Просмотр коммита HEAD
git show HEAD
git log -1 HEAD
```

## Нотация HEAD~N и HEAD^

Git поддерживает относительные ссылки от HEAD:

```bash
# HEAD~1 — один коммит назад
git show HEAD~1
git diff HEAD~1

# HEAD~3 — три коммита назад
git log HEAD~3..HEAD

# HEAD^ — родительский коммит (аналог HEAD~1)
git show HEAD^

# HEAD^2 — второй родитель (при merge коммите)
# Merge коммит имеет двух родителей:
git show HEAD^   # первый родитель (основная ветка)
git show HEAD^2  # второй родитель (сливаемая ветка)

# Комбинирование
git show HEAD~2^  # родитель двух коммитов назад
```

Разница `~` и `^`:
- `~N` — N шагов по первому родителю
- `^N` — N-й родитель коммита (для merge коммитов)

## Перемещение HEAD

```bash
# Переместить HEAD на другую ветку
git switch main
git checkout main

# Переместить HEAD на конкретный коммит (detached HEAD)
git checkout a1b2c3d
git switch --detach a1b2c3d

# Переместить HEAD на тег
git checkout v1.0

# Переместить HEAD на N коммитов назад
git checkout HEAD~3

# Переместить ветку (не HEAD) на другой коммит
git branch -f main HEAD~3
```

## Detached HEAD: что это и почему опасно

Detached HEAD возникает когда HEAD указывает напрямую на коммит, а не на ветку:

```bash
# Перейти в detached HEAD
git checkout a1b2c3d

# Предупреждение Git:
# You are in 'detached HEAD' state. You can look around,
# make experimental changes and commit them, and you can
# discard any commits you make in this state without impacting
# any branches by switching back to a branch.

# Проверить состояние
git status
# HEAD detached at a1b2c3d
```

**Опасность**: коммиты в detached HEAD режиме «висят в воздухе» — они не принадлежат никакой ветке. После переключения на ветку они станут недоступными и будут удалены сборщиком мусора.

```bash
# В detached HEAD можно делать коммиты
git commit -m "Experimental change"
# Но если переключиться на ветку — этот коммит потеряется

# Для сохранения коммитов — создать ветку
git switch -c new-branch  # создаёт ветку с текущего места
```

## Возврат из Detached HEAD

```bash
# Способ 1: переключиться на существующую ветку
git switch main
git checkout main

# Способ 2: создать новую ветку из detached HEAD
git switch -c my-new-branch

# Способ 3: если сделали коммиты и хотите их сохранить
# Запомнить хеш последнего коммита
git log --oneline -1  # например: a1b2c3d
git switch main
git cherry-pick a1b2c3d  # применить коммит к ветке
```

## ORIG_HEAD, MERGE_HEAD, FETCH_HEAD

Git использует несколько специальных файлов HEAD:

```bash
# ORIG_HEAD — HEAD до опасной операции (merge, rebase, reset)
# Используется для отмены
git reset --hard ORIG_HEAD  # отменить merge/rebase

# MERGE_HEAD — коммит сливаемой ветки (во время merge)
cat .git/MERGE_HEAD  # существует только во время merge

# FETCH_HEAD — последний fetchнутый коммит
cat .git/FETCH_HEAD
git log FETCH_HEAD  # что было получено последним fetch
```

## Практические примеры

```bash
# Посмотреть изменения последних 3 коммитов
git diff HEAD~3..HEAD

# Вернуться к предыдущей версии файла
git restore --source=HEAD~2 src/main.js

# Создать новую ветку от конкретного коммита
git switch -c hotfix HEAD~5

# Посмотреть на какой ветке HEAD
git branch --points-at HEAD

# Проверить сколько коммитов от HEAD до main
git log HEAD..main --oneline | wc -l
```

## Часто задаваемые вопросы

**Что означает "HEAD detached at"?** HEAD не указывает на ветку, а напрямую на коммит. Это нормально для просмотра (git checkout тег/хеш), но коммиты в этом состоянии не сохранятся после переключения на ветку.

**Как потерял коммиты при detached HEAD?** После `git switch main` коммиты стали недоступны, потому что они не принадлежат ветке. Можно восстановить через `git reflog` — история всех HEAD перемещений.

**Чем HEAD~1 отличается от HEAD^?** Для обычных коммитов — идентично: оба означают родительский коммит. Разница проявляется у merge коммитов: `HEAD^` и `HEAD^2` — первый и второй родители слияния.

**Можно ли сделать коммит в detached HEAD?** Да. Но эти коммиты не принадлежат ветке. Для их сохранения — создайте ветку: `git switch -c new-branch`.

## Заключение

HEAD — это указатель на текущее положение в истории Git. Обычно указывает на ветку, которая указывает на последний коммит. Detached HEAD возникает при прямом checkout коммита/тега. Нотация `HEAD~1`, `HEAD~3`, `HEAD^` позволяет ссылаться на предыдущие коммиты. Для восстановления после потери коммитов используйте [git reflog]({{< relref "git-reflog" >}}).
