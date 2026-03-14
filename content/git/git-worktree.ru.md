---
title: "git worktree: работа с несколькими ветками одновременно"
description: "Руководство по git worktree. Работа с несколькими ветками одновременно без stash. Создание, управление и удаление worktrees. Практические сценарии."
date: 2026-02-02
lastmod: 2026-02-02
draft: false
slug: "git-worktree"
keywords: ["git worktree", "несколько веток git одновременно", "git worktree add", "git worktree что делает", "git worktree list", "несколько рабочих директорий git"]
tags: ["git", "advanced"]
categories: ["git"]
---

`git worktree` позволяет иметь несколько рабочих директорий, связанных с одним репозиторием. Это решает проблему когда нужно срочно переключиться на другую задачу, но текущая работа незакончена и не хочется делать stash.

## Проблема без worktree

```bash
# Сценарий: разрабатываете feature, пришёл срочный баг
git status
# Changes not staged for commit:
#   modified:   src/feature.js

# Нельзя просто git switch hotfix — есть незакоммиченные изменения
# Нужно либо:
git stash          # временно сохранить
git switch hotfix  # переключиться
# ... исправить баг ...
git switch feature
git stash pop      # восстановить

# Или: git worktree позволяет иметь обе ветки одновременно
```

## Создание worktree

```bash
# Создать worktree для ветки hotfix в папке ../hotfix
git worktree add ../hotfix hotfix

# Создать worktree и новую ветку
git worktree add -b feature/new-login ../new-login main

# Создать worktree для конкретного коммита
git worktree add --detach ../review a1b2c3d

# Синтаксис: git worktree add <path> [<branch>]
# <path> — папка для нового worktree
# <branch> — ветка (должна существовать)
```

После этого появляется новая папка `../hotfix` с полноценным рабочим деревом — можно редактировать файлы, запускать тесты, делать коммиты.

## Список worktrees

```bash
git worktree list
# /home/user/project           a1b2c3d [main]
# /home/user/hotfix            e5f6a7b [hotfix]
# /home/user/new-login         c3d4e5f [feature/new-login]
```

## Работа в worktree

```bash
# Перейти в hotfix worktree
cd ../hotfix

# Обычная работа с Git
git status
git add src/bugfix.js
git commit -m "fix: resolve null pointer in login"
git push origin hotfix

# Вернуться в основной репозиторий
cd ../project
# Продолжить работу над feature — всё на месте, stash не нужен
```

Все worktrees связаны с одним `.git` — изменения в одном видны в другом через `git fetch` и `git log`.

## Удаление worktree

```bash
# Удалить worktree
git worktree remove ../hotfix

# Принудительно (если есть незакоммиченные изменения)
git worktree remove --force ../hotfix

# Очистить ссылки на удалённые worktrees
git worktree prune
```

## Практические сценарии

**Срочный hotfix во время разработки:**

```bash
# В основном репозитории идёт разработка feature
git worktree add -b hotfix/critical-bug ../hotfix-work main

cd ../hotfix-work
# Исправить баг
git commit -am "fix: critical security patch"
git push origin hotfix/critical-bug
# Создать PR/MR

# Вернуться к feature
cd ../project
# Ничего не потеряно, stash не нужен
```

**Сравнение двух версий:**

```bash
# Открыть две версии одновременно
git worktree add ../v1-compare v1.0
git worktree add ../v2-compare v2.0

# Сравнить файлы напрямую
diff ../v1-compare/src/api.js ../v2-compare/src/api.js

# Или запустить оба приложения на разных портах
cd ../v1-compare && PORT=3001 npm start &
cd ../v2-compare && PORT=3002 npm start &
```

**Code review без переключения:**

```bash
# Проверить чужую ветку не переключаясь
git fetch origin
git worktree add ../review-pr-123 origin/feature/user-auth

cd ../review-pr-123
# Посмотреть код, запустить тесты
npm test

# После ревью
cd ../project
git worktree remove ../review-pr-123
```

## Ограничения worktree

```
- Одну ветку нельзя использовать в нескольких worktrees одновременно
- Rebase и merge в одном worktree блокирует другие для этой ветки
- Bare репозитории работают немного иначе
- Hook'и работают в контексте конкретного worktree
```

```bash
# Попытка checkout уже используемой ветки:
git worktree add ../other-feature feature/auth
# fatal: 'feature/auth' is already checked out at '/home/user/project'
```

## Управление worktrees

```bash
# Список всех worktrees
git worktree list

# Список с детальной информацией
git worktree list --porcelain

# Переместить worktree в другую папку
git worktree move ../hotfix ../bugfixes/hotfix-2024

# Заблокировать worktree (не удалять при prune)
git worktree lock ../hotfix --reason "In use for critical bug"

# Разблокировать
git worktree unlock ../hotfix

# Очистить информацию об удалённых worktrees
git worktree prune
```

## Часто задаваемые вопросы

**Чем worktree отличается от git stash?** Stash временно сохраняет изменения в стеке. Worktree — полноценная рабочая копия в отдельной папке. В worktree можно работать параллельно, запускать тесты, редактировать файлы без переключения контекста.

**Чем worktree отличается от второго клона репозитория?** Оба способа дают несколько копий. Но worktrees используют один объектный хранилище `.git/objects` — не дублируют данные. Клон — полностью независимая копия.

**Можно ли делать коммиты во всех worktrees?** Да, в каждом worktree независимо. Но одна и та же ветка не может быть в двух worktrees.

**Что происходит с worktrees при удалении папки вручную?** Git не знает об удалении. Выполните `git worktree prune` для очистки ссылок на несуществующие worktrees.

## Сравнение worktree, stash и второго клона

Три способа работать с несколькими ветками:

```bash
# Способ 1: git stash (временное сохранение)
# Текущее состояние:
#   Ветка: feature/auth (незакончена)
#   Файлы: изменены но не закоммичены

# Сценарий: срочный баг
git stash
# Сохраняет изменения в стек

git switch hotfix
# Исправляем баг
git switch feature/auth
git stash pop
# Восстанавливаем из стека

# Минусы: работает только с текущей веткой, может быть сложно с несколькими stash'ами
# Плюсы: просто, не требует дополнительных папок

# Способ 2: git worktree (несколько рабочих директорий)
git worktree add -b hotfix ../hotfix main
# cd ../hotfix
# (работаем на hotfix в отдельной папке)
# cd ../project
# (feature/auth всё ещё открыта в исходной папке)

# Плюсы: параллельная работа, можно запускать тесты одновременно
# Минусы: требует несколько папок, одну ветку нельзя использовать дважды

# Способ 3: второй клон репозитория
git clone /path/to/original ../another-copy
# cd ../another-copy
# Полностью независимая копия

# Плюсы: полная независимость, можно на разные версии коммитить
# Минусы: дублирует все файлы и историю, расходует больше места на диске

# Сравнение:
# stash:        быстро, простое, одна ветка одновременно
# worktree:     параллельная работа, одно хранилище .git
# clone:        полная независимость, больше места
```

## Использование worktree для запуска тестов

Проверить что тесты проходят на разных ветках одновременно:

```bash
# В основной папке разрабатываете feature
# Нужно убедиться что main ветка всё ещё проходит тесты

git worktree add -b test-main ../test-main main
# cd ../test-main
npm install
npm test
# Если тесты падают — может быть конфликт

# В другой папке продолжаете feature
# cd ../project
npm test
# Запускаете свои тесты одновременно

# Результат: знаете что основная ветка работает, а ваша feature — тоже
```

## Переименование и перемещение worktree

```bash
# Переместить worktree в другую папку
git worktree move ../hotfix ../bugfixes/hotfix-2024

# После перемещения можно спокойно работать в новой папке
cd ../bugfixes/hotfix-2024

# Удалить из старой папки (если осталась)
rm -rf ../hotfix

# Проверить что перемещение сработало
git worktree list
```

## Работа с bare репозиториями (для серверов)

На серверах часто используют bare репозитории:

```bash
# Создать bare репозиторий
git init --bare project.git

# Добавить worktree в bare репозитории
git -C project.git worktree add ../project-web main
git -C project.git worktree add ../project-api develop

# Теперь на сервере есть несколько рабочих директорий
# из одного bare репозитория
```

## Заблокированные worktrees

Иногда нужно защитить worktree от случайного удаления:

```bash
# Заблокировать worktree
git worktree lock ../hotfix --reason "In use for critical bug fix"

# Список с информацией о блокировке
git worktree list --verbose

# Попытка удалить заблокированный worktree
git worktree remove ../hotfix
# fatal: cannot remove locked working tree

# Разблокировать
git worktree unlock ../hotfix

# Теперь можно удалить
git worktree remove ../hotfix
```

## Очистка старых worktrees (git worktree prune)

После удаления worktrees остаются ссылки:

```bash
# Список всех worktrees (включая удалённые)
git worktree list --porcelain

# Если папка worktree была удалена вручную (rm -rf)
# Git всё ещё знает про неё

# Очистить ссылки на удалённые worktrees
git worktree prune

# Теперь git worktree list покажет только существующие
```

## Ограничения которые важно знать

```
1. Одну ветку нельзя checkout в двух worktrees одновременно
   git worktree add ../other feature/auth  # если уже на feature/auth
   # fatal: 'feature/auth' is already checked out

2. Git операции блокируют друг друга
   Если в одном worktree idёт rebase — другие не могут мержить эту ветку

3. Hooks работают в контексте конкретного worktree
   pre-commit hook в ../hotfix — не видит changes в ../project

4. Общее .git в основной папке
   Если удалить основную папку — все worktrees сломаются

5. Некоторые операции требуют осторожности
   git gc --aggressive может временно заблокировать все worktrees
```

## Продвинутые сценарии

**Работа на нескольких модулях одновременно:**
```bash
# Monorepo структура
project/
  ├─ apps/web
  ├─ apps/api
  └─ packages/shared

# Создать worktrees для разных веток
git worktree add ../frontend-feature feature/frontend-v2
git worktree add ../api-feature feature/api-improvements

# Работать над frontend и API одновременно
# Тестировать интеграцию параллельно
```

**Bisect для отладки в отдельном worktree:**
```bash
# git bisect требует переключать ветки часто
# Это медленно в основном worktree

# Создать отдельный для bisect
git worktree add --detach ../bisect-debug main

cd ../bisect-debug
git bisect start
git bisect bad
git bisect good v1.0.0
# Быстрый bisect в отдельной папке

# Основная работа продолжается в главной папке
```

## Часто задаваемые вопросы

**Чем worktree отличается от git stash?** Stash временно сохраняет изменения в стеке. Worktree — полноценная рабочая копия в отдельной папке. В worktree можно работать параллельно, запускать тесты, редактировать файлы без переключения контекста.

**Чем worktree отличается от второго клона репозитория?** Оба способа дают несколько копий. Но worktrees используют один объектный хранилище `.git/objects` — не дублируют данные. Клон — полностью независимая копия с собственным .git.

**Можно ли делать коммиты во всех worktrees?** Да, в каждом worktree независимо. Но одна и та же ветка не может быть в двух worktrees.

**Что происходит с worktrees при удалении папки вручную?** Git не знает об удалении. Выполните `git worktree prune` для очистки ссылок на несуществующие worktrees.

**Может ли быть конфликт между worktrees?** Нет конфликта на уровне Git. Но если вы работаете в одном worktree на hotfix и в другом на merge той же ветки — результат может быть неожиданным. Избегайте одновременных операций с одной веткой.

**Нужно ли синхронизировать worktrees?** Нет, `git fetch` в одном автоматически обновляет информацию для всех так как они используют один .git.

**Как удалить все worktrees и оставить только основной репозиторий?**
```bash
git worktree list
git worktree remove <path1>
git worktree remove <path2>
git worktree prune
```

## Заключение

`git worktree` — мощный инструмент для параллельной работы с несколькими ветками без stash. Добавление: `git worktree add <path> <branch>`. Основной сценарий: срочный hotfix пока идёт разработка feature. Удаление: `git worktree remove <path>`. Не забывайте `git worktree prune` для очистки ссылок. Используйте worktrees когда нужна параллельная работа на разных ветках, git stash когда нужно временно сохранить изменения. Для обычных переключений между ветками — [git stash]({{< relref "git-stash" >}}).
