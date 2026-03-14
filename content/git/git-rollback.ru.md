---
title: "git rollback: как откатиться к предыдущему коммиту в Git"
description: "Как сделать rollback в Git: git revert (безопасно), git reset --hard (локально), откат файла, откат на неделю назад. Примеры для всех сценариев."
date: 2026-01-12
lastmod: 2026-03-14
draft: false
slug: "git-rollback"
keywords: ["git rollback", "git rollback commit", "откат git", "git откат к коммиту", "git вернуться к предыдущей версии"]
tags: ["git", "beginner"]
categories: ["git"]
---

В Git нет встроенной команды `git rollback`. Это часто сбивает с толку новичков, которые ищут эту команду в документации и не находят. Однако «откат» в Git реализуется несколькими способами в зависимости от того, что именно вы хотите сделать.

Эта статья служит навигационным руководством: мы разберём все возможные сценарии отката, покажем какая команда подходит для каждого случая, и дадим практические примеры.

Главный вывод: слово «откат» в контексте Git может означать совершенно разные вещи, и вы должны выбрать правильную команду для своей задачи.

## Что означает «откат» в Git: три разных задачи

Когда разработчик говорит «мне нужен откат», часто имеется в виду одна из трёх совершенно разных операций:

### 1. Посмотреть версию файла из старого коммита (без изменения истории)

Вы просто хотите посмотреть как выглядел файл месяц назад. Это не меняет ничего в текущей ветке.

```bash
git show abc1234:path/to/file.js
# или
git checkout abc1234 -- path/to/file.js
```

### 2. Отменить последние несколько коммитов (вернуться к старому состоянию)

Вы хотите чтобы проект вернулся в состояние как он был несколько коммитов назад. Это может быть `git reset` или `git revert`. Подробнее о reset см. в {{< relref "git-reset" >}}.

### 3. Отменить изменения одного конкретного коммита (но оставить остальные)

Есть плохой коммит в середине истории. Вы хотите отменить именно его изменения, но остальные коммиты остаются.

```bash
git revert abc1234
```

Давайте разберём каждый случай подробно.

## Откат всего проекта к определённому коммиту

Самый частый сценарий: нужно вернуть весь проект в состояние из определённого коммита.

### Шаг 1: найдите нужный коммит

```bash
# Посмотрите последние коммиты
git log --oneline

# Или посмотрите коммиты за последнюю неделю
git log --oneline --since="1 week ago"

# Или коммиты по определённому файлу
git log --oneline -- src/auth.js
```

Пример вывода:
```
d7f8e9a Remove feature X
c4b3a2d Add feature X
b1c2d3e Refactor database
a1b2c3d Initial commit
```

### Шаг 2: выберите между git revert и git reset

**Если коммит локальный (не запушен):**
```bash
# Откатиться к коммиту b1c2d3e, удалив всё что после
git reset --hard b1c2d3e

# Или сохранить изменения
git reset --mixed b1c2d3e
```

**Если коммит уже в удалённом репо (на GitHub и т.д.):**
```bash
# Отменить последний коммит
git revert HEAD

# Запушить отмену
git push
```

### Пример: полный откат проекта

```bash
$ git log --oneline
d7f8e9a Broken deploy (HEAD -> main)
c4b3a2d Add feature X
b1c2d3e Refactor database
a1b2c3d Initial commit

# Коммит d7f8e9a сломал всё, но он уже в продакшене
# Используем git revert для безопасного отката

$ git revert d7f8e9a
# Откроется редактор, можно оставить сообщение по умолчанию или отредактировать
# Сохраняем и выходим

$ git log --oneline
8x9y0z1 Revert "Broken deploy" (HEAD -> main)
d7f8e9a Broken deploy
c4b3a2d Add feature X
b1c2d3e Refactor database

$ git push
# Ваша отмена отправлена в репо
```

## Откат с сохранением файлов в рабочей директории

Если нужно откатиться, но при этом сохранить возможность пересмотреть файлы перед новым коммитом:

```bash
# Откатить последние 3 коммита, оставив изменения в рабочей директории
git reset --mixed HEAD~3

# Или используйте просто git reset (--mixed по умолчанию)
git reset HEAD~3

# Проверите что изменилось
git status
git diff

# Можете отредактировать файлы и сделать новый коммит
git add .
git commit -m "New better commit"
```

**Практический пример:**
```bash
# Вы сделали 3 коммита но они не идеальны
$ git log --oneline
1f2e3d4 Update tests (HEAD -> feature-branch)
5a6b7c8 Refactor auth module
9d8e7f6 Add new auth logic
c4b3a2d Previous stable commit

# Откатываем последние 3
$ git reset HEAD~3

$ git status
On branch feature-branch
Changes not staged for commit:
  modified:   src/auth.js
  modified:   src/auth.test.js
  modified:   tests/integration.test.js

# Теперь переделываем всё в один хороший коммит
$ git add .
$ git commit -m "Add improved authentication system with tests"

$ git log --oneline
a1b2c3d Add improved authentication system with tests (HEAD -> feature-branch)
c4b3a2d Previous stable commit
```

## Откат одного файла к старой версии

Иногда нужно вернуть один файл к предыдущему состоянию, остальные файлы трогать не нужно.

```bash
# Вернуть файл к версии из предыдущего коммита
git restore --source=HEAD~1 -- path/to/file.js

# Или к версии из конкретного коммита
git restore --source=abc1234 -- path/to/file.js

# Старый синтаксис (ещё работает)
git checkout abc1234 -- path/to/file.js
```

После этой команды файл будет в staging area и готов к коммиту:

```bash
$ git status
On branch main
Changes to be committed:
  modified:   src/config.js

$ git commit -m "Restore config.js to previous version"
```

### Пример: откат одного файла в продакшене

```bash
# Нужно откатить конфиг файл который сломал систему
$ git log --oneline -- config/production.js
a1b2c3d Update production config (HEAD -> main)
f4e5d6c Fix database URL
d7c8e9f Add cache config
8x9y0z1 Initial config

# Вернуть к рабочей версии
$ git restore --source=f4e5d6c -- config/production.js

$ git status
On branch main
Changes to be committed:
  modified:   config/production.js

$ git commit -m "Revert production config to stable version"

$ git push
```

## Откат через временный переход к старому коммиту

Иногда нужно просто посмотреть как выглядел проект в определённый момент времени. Git позволяет это сделать:

```bash
# Перейти к состоянию из коммита abc1234
git checkout abc1234

# Git перейдёт в detached HEAD состояние
# Вы можете смотреть файлы, запускать старую версию кода, и т.д.

# Если нашли что-то важное, можете создать новую ветку
git switch -c investigate-old-version

# Или вернуться в главную ветку
git switch main
```

**Полный пример:**
```bash
$ git log --oneline
d7f8e9a Current version (HEAD -> main)
c4b3a2d Previous version
b1c2d3e Version two weeks ago

# Хотим посмотреть как выглядел проект две недели назад
$ git checkout b1c2d3e
$ git status
HEAD detached at b1c2d3e

# Смотрим, запускаем, тестируем старую версию

# Нашли важный файл который был потерян, создаём ветку
$ git switch -c recover-old-feature

# Там мы можем сделать коммиты и потом смержить в main

# Или если ничего не нужно, просто возвращаемся назад
$ git switch main
$ git branch -D recover-old-feature
```

## Откат push в production: безопасно и быстро

Самый критичный сценарий: коммит попал в production и что-то сломал. Нужно откатиться быстро и безопасно.

### Правильный способ: git revert

```bash
# Если сломан последний коммит
git revert HEAD

# Если нужно отменить последние 3 коммита
git revert HEAD~2..HEAD

# Эта команда создаст два новых коммита отмены
# История сохранится, все видят что произошло

# Запушиваем отмену
git push
```

### Опасный способ (НЕ РЕКОМЕНДУЕТСЯ): git reset --hard

```bash
# ОПАСНО! Только если вы совсем отчаялись и уверены
git reset --hard HEAD~1
git push --force

# Это сломает историю для всех у кого уже есть этот коммит
# Все команды будут ругаться
```

### Полный сценарий production откатывания

```bash
# В 15:30 замечаем что в production проблема
# Проверяем последний коммит
$ git log --oneline -n 5
a1b2c3d Bad deploy - payment broken (HEAD -> main)
f4e5d6c Add new payment processing
d7c8e9f Refactor checkout

# Ясно что a1b2c3d это проблема
# Отменяем его через git revert

$ git revert a1b2c3d
# Редактор откроется, оставляем сообщение или пишем свое
# В моём случае я напишу "Revert broken payment changes"

# Сохраняем редактор, выходим

$ git log --oneline
8x9y0z1 Revert "Bad deploy - payment broken"
a1b2c3d Bad deploy - payment broken
f4e5d6c Add new payment processing

# Проверяем что код выглядит правильно
$ git status
On branch main
nothing to commit, working tree clean

# Запушиваем
$ git push

# Ждём deploy
# Production восстановлена!
```

## Таблица сравнения: команды по сценариям

| Задача | Команда | Примечание |
|--------|---------|-----------|
| Откатить локальный коммит, удалить всё | `git reset --hard HEAD~1` | Безвозвратно! |
| Откатить локальный коммит, сохранить файлы | `git reset HEAD~1` | Файлы остаются в рабочей директории |
| Откатить запушенный коммит | `git revert HEAD` | Создаёт новый коммит отмены |
| Откатить несколько запушенных | `git revert HEAD~2..HEAD` | Два новых коммита отмены |
| Откатить один файл к старой версии | `git restore --source=abc1234 -- file.js` | Файл в staging, готов к коммиту |
| Посмотреть старую версию проекта | `git checkout abc1234` | Detached HEAD, можно смотреть |
| Откатить 5 последних коммитов | `git reset HEAD~5` | Все файлы в рабочей директории |
| Откатить к конкретному коммиту | `git reset abc1234` | Всё что после отменяется |

## FAQ: Частые вопросы о откате

**Q: Есть ли команда git rollback?**

A: Нет официальной команды `git rollback`. Это просто термин, который разные люди понимают по-разному. Для отката используйте `git reset` (локально) или `git revert` (для запушенных коммитов).

**Q: Какую команду использовать для откатывания в production?**

A: **Всегда** используйте `git revert` для production коммитов. Это создаёт новый коммит, сохраняет историю и безопасно. Никогда не используйте `git reset --hard` с `git push --force` в production.

**Q: Как откатиться на неделю назад?**

A: Сначала найдите коммит:
```bash
git log --oneline --since="1 week ago"
```
Потом либо откатитесь на конкретный коммит, либо переверните нужные коммиты:
```bash
git revert abc1234  # Если нужно сохранить историю
```

**Q: Как восстановить удалённый коммит?**

A: Используйте `git reflog`:
```bash
git reflog
git reset --hard abc1234  # Коммит которы хотите восстановить
```

**Q: В чём разница между откатом и revert?**

A: **Откат** (reset) переписывает историю — удаляет коммиты полностью. **Revert** создаёт новый коммит отмены — история остаётся. Используйте reset для локальных, revert для опубликованных.

## Заключение

Git откат может быть простым или сложным в зависимости от ситуации:

- **Локальный коммит**: `git reset`
- **Опубликованный коммит**: `git revert`
- **Один файл**: `git restore --source=HASH`
- **Посмотреть старое состояние**: `git checkout HASH`

Помните: для production **всегда** используйте `git revert`. Это безопаснее и понятнее для команды.

Для более глубокого понимания откатывания см. {{< relref "git-undo-last-commit" >}}, {{< relref "git-revert" >}}, {{< relref "git-reset" >}} и {{< relref "git-reflog" >}}.
