---
title: "git status: что это и как читать вывод команды"
description: "Подробное руководство по git status. Как читать вывод, состояния файлов, компактный формат, проверка перед коммитом. Примеры использования."
date: 2026-01-30
lastmod: 2026-01-30
draft: false
slug: "git-status"
keywords: ["git status", "список отслеживаемых файлов git", "git статус репозитория", "git status что делает", "git status -s", "git состояние файлов"]
tags: ["git", "beginner"]
categories: ["git"]
---

`git status` — одна из самых часто используемых команд в Git. Она показывает текущее состояние рабочей директории и индекса (staging area): какие файлы изменены, какие подготовлены к коммиту, какие не отслеживаются. Это первая команда, которую стоит запускать перед любым коммитом.

## Базовое использование

```bash
git status
```

Вывод при чистом рабочем дереве (нет изменений):

```
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

Вывод при наличии изменений:

```
On branch main
Your branch is ahead of 'origin/main' by 1 commit.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   docs/setup.md
        modified:   README.md

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   src/main.js

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        temp.log
        secrets.env
```

## Интерпретация разделов вывода

Вывод `git status` состоит из трёх разделов:

**Changes to be committed (зелёный):**
Файлы в индексе (staging area) — готовы к коммиту. `git add` уже был выполнен.

**Changes not staged for commit (красный):**
Файлы изменены на диске, но `git add` ещё не был выполнен. Не попадут в следующий коммит.

**Untracked files (красный):**
Новые файлы, которые Git вообще не отслеживает. Нужно явно добавить через `git add`.

## Состояния файлов

```
new file    — новый файл добавлен через git add (появится в staged)
modified    — существующий файл изменён
deleted     — файл удалён
renamed     — файл переименован (через git mv)
untracked   — файл не под контролем Git
```

## Компактный формат (--short / -s)

```bash
git status -s
# или
git status --short
```

Вывод:

```
M  README.md      — modified (в индексе)
 M src/main.js    — modified (не в индексе)
A  docs/setup.md  — added (новый файл в индексе)
?? temp.log       — untracked
!! node_modules/  — ignored (только с --ignored)
```

Первый символ — состояние в индексе. Второй — состояние в рабочей директории.

Таблица символов:

```
' ' — не изменён
M   — изменён
A   — добавлен
D   — удалён
R   — переименован
C   — скопирован
U   — обновлён (unmerged)
?   — untracked
!   — ignored
```

## Информация о ветке

```bash
# Краткая информация о ветке
git status -sb
# ## main...origin/main [ahead 1]

# Расшифровка:
# ## main               — текущая ветка
# ...origin/main        — tracking branch
# [ahead 1]             — 1 коммит, не запушенный на remote
# [behind 2]            — 2 коммита на remote, не подтянутых локально
# [ahead 1, behind 2]   — оба направления расходятся
```

## Просмотр статуса конкретного файла или папки

```bash
# Статус только src/ папки
git status src/

# Статус конкретного файла
git status README.md
```

## Проверка статуса перед коммитом

Хороший рабочий процесс перед каждым коммитом:

```bash
# Шаг 1: Посмотреть что изменено
git status

# Шаг 2: Просмотреть детали изменений
git diff         # не в индексе
git diff --staged # в индексе

# Шаг 3: Добавить нужные файлы
git add src/main.js
git add README.md

# Шаг 4: Снова проверить что попадёт в коммит
git status

# Шаг 5: Закоммитить
git commit -m "Описание изменений"
```

## Часто встречаемые ситуации

**Файл в обоих разделах:**

```
Changes to be committed:
        modified:   src/auth.js

Changes not staged for commit:
        modified:   src/auth.js
```

Это значит: часть изменений добавлена через `git add`, после чего файл изменялся ещё раз. В коммит попадёт только та версия, которая была при `git add`.

**Конфликты при merge:**

```
Unmerged paths:
  (use "git add <file>..." to mark resolution)
        both modified:   src/config.js
```

Файл в состоянии конфликта — нужно разрешить конфликт и `git add`.

## Проверка игнорируемых файлов

```bash
# Показать также игнорируемые файлы
git status --ignored

# Вывод добавит раздел:
# Ignored files:
#         node_modules/
#         .env
#         *.log
```

## Часто задаваемые вопросы

**Какая разница между "Changes to be committed" и "Changes not staged"?** Первое — файлы в staging area (index), которые попадут в следующий коммит. Второе — файлы изменены на диске, но `git add` ещё не был выполнен, они в коммит не попадут.

**Как убрать файл из staging area?** `git restore --staged <file>` или `git reset HEAD <file>` (старый синтаксис). Файл останется изменённым, просто не попадёт в коммит.

**Что делать если git status показывает много изменений?** Используйте `-s` для компактного вывода. Если нужно проверить что именно изменилось — `git diff`.

**Как сделать git status быстрее в большом репозитории?** `git status -u no` отключает показ untracked файлов. Также помогает наличие `.gitignore` для исключения большого числа файлов.

**Почему git status показывает изменения в файле, хотя я ничего не менял?** Возможная причина — разные line endings (CRLF vs LF) или изменение прав доступа на файл. Проверьте настройку `core.autocrlf` и `core.fileMode`.

## Заключение

`git status` — это базовая команда для понимания текущего состояния репозитория. Запускайте её перед каждым `git add` и `git commit`. Для детального просмотра изменений используйте [git diff]({{< relref "git-diff" >}}). Для истории коммитов — [git log]({{< relref "git-log" >}}).
