---
title: "git rm: удаление файлов из Git репозитория"
description: "Команда git rm: удаление файлов из Git, git rm --cached, восстановление удалённых файлов, правильный процесс."
date: 2026-01-28
lastmod: 2026-01-28
draft: false
slug: "git-rm"
keywords:
  - git rm file
  - git add rm
  - удалить файл из git
  - как удалить файл git
tags:
  - git
  - beginner
categories:
  - Git
aliases: []
---

Удаление файла из Git-репозитория требует специальной команды — `git rm`. Просто удалить файл через `rm` или проводник недостаточно: Git продолжит его отслеживать и будет показывать в статусе как «deleted». Команда `git rm` делает два дела сразу: удаляет файл и регистрирует это удаление в индексе для последующего коммита.

## Три способа удалить файл — правильный и неправильные

Чтобы понять зачем нужна `git rm`, посмотрим на разницу между подходами:

**Неправильно — просто удалить файл:**

```bash
rm old-file.js
git status
# Changes not staged for commit:
#   deleted:    old-file.js
```

Файл удалён с диска, но Git всё ещё его отслеживает. Придётся дополнительно делать `git add -u` или `git add old-file.js`, чтобы зарегистрировать удаление.

**Правильно — использовать git rm:**

```bash
git rm old-file.js
git status
# Changes to be committed:
#   deleted:    old-file.js

git commit -m "Remove old-file.js"
```

Один шаг вместо двух: файл удалён с диска и удаление уже в индексе.

**Удалить из Git но сохранить локально:**

```bash
git rm --cached .env
# Файл остаётся на диске, но Git перестанет его отслеживать
```

Этот вариант разберём подробнее ниже.

## Базовый синтаксис

```bash
# Удалить один файл
git rm файл.js

# Удалить несколько файлов
git rm файл1.js файл2.js

# Удалить все файлы с расширением .log
git rm *.log

# Удалить папку со всем содержимым (рекурсивно)
git rm -r old-directory/
```

После `git rm` нужно сделать коммит — само удаление ещё не сохранено в истории:

```bash
git rm old-file.js
git commit -m "Remove old-file.js: больше не используется"
```

## git rm --cached: убрать из Git, сохранить файл

Это один из самых полезных вариантов `git rm`. Флаг `--cached` удаляет файл из отслеживания Git, но оставляет его на диске.

Классический сценарий: файл с секретами или конфигурацией случайно попал в репозиторий:

```bash
# .env с паролями случайно закоммичен
git rm --cached .env

# Добавить в .gitignore чтобы не повторилось
echo ".env" >> .gitignore

# Закоммитить оба изменения
git add .gitignore
git commit -m "Remove .env from tracking, add to .gitignore"
```

Теперь `.env` остаётся у тебя локально (со всеми паролями), но в репозитории его нет, и следующие коммиты его игнорируют.

Другой сценарий — удалить из репозитория папку с зависимостями, которая туда случайно попала:

```bash
# node_modules случайно закоммичен
git rm -r --cached node_modules/

# Добавить в .gitignore
echo "node_modules/" >> .gitignore

git add .gitignore
git commit -m "Remove node_modules from tracking"
```

## Проверка перед удалением

Если хочешь сначала посмотреть что будет удалено — используй флаг `-n` (dry run):

```bash
# Посмотреть что удалится без реального удаления
git rm -n *.log
# rm 'debug.log'
# rm 'error.log'
# rm 'access.log'

# Убедившись — удалить реально
git rm *.log
```

## Принудительное удаление

Если файл изменён, но изменения не закоммичены, `git rm` откажет:

```bash
$ git rm important-file.js
error: the following files have local modifications:
    important-file.js
(use --cached to keep the file, or -f to force removal)
```

Флаг `-f` (force) отменяет эту защиту:

```bash
# ⚠️ Внимание: незакоммиченные изменения будут потеряны безвозвратно
git rm -f important-file.js
```

> ⚠️ **Осторожно:** принудительное удаление уничтожает незакоммиченные изменения. Убедись, что они тебе действительно не нужны.

## Восстановление удалённого файла

Удаление файла через `git rm` и коммит — не катастрофа. Файл можно восстановить из истории:

```bash
# Посмотреть историю удалённого файла
git log --all -- old-file.js

# Восстановить файл из конкретного коммита
git restore --source=<commit-hash> old-file.js

# Восстановить из коммита до удаления (HEAD~1 = предыдущий коммит)
git restore --source=HEAD~1 old-file.js

# Альтернатива (старый синтаксис до Git 2.23)
git checkout <commit-hash> -- old-file.js
```

После восстановления файл появится в рабочей директории. Нужно сделать `git add` и `git commit` чтобы вернуть его в актуальную ветку.

## Удаление файлов, уже удалённых через rm

Если ты уже удалил файлы обычным `rm` и хочешь зарегистрировать это в Git:

```bash
# Удалили файлы через rm, git status показывает:
# Changes not staged for commit:
#   deleted:    file1.js
#   deleted:    file2.js

# Способ 1: добавить все удаления в индекс
git add -u

# Способ 2: добавить конкретный удалённый файл
git add file1.js

# Способ 3: добавить всё (включая новые файлы)
git add -A

# После этого коммитить
git commit -m "Remove obsolete files"
```

## Правильный рабочий процесс удаления

```bash
# 1. Проверить текущее состояние
git status

# 2. Посмотреть что удалится (dry run)
git rm -n old-code/

# 3. Удалить
git rm -r old-code/

# 4. Проверить что в индексе
git status
# Changes to be committed:
#   deleted:    old-code/helper.js
#   deleted:    old-code/utils.js

# 5. Посмотреть diff
git diff --cached

# 6. Закоммитить с понятным сообщением
git commit -m "Remove old-code/: заменено на src/utils/"

# 7. Убедиться что файлов нет в актуальной версии
git ls-files | grep old-code
# (пустой вывод — файлов нет)
```

## git rm и .gitignore

Важный нюанс: если файл уже отслеживается Git, добавление его в `.gitignore` не остановит отслеживание. Сначала нужно удалить его из индекса:

```bash
# Неправильный порядок: добавил в .gitignore, но файл всё равно отслеживается
echo "config.local.js" >> .gitignore
git status
# Changes not staged for commit:
#   modified:    config.local.js   ← всё ещё отслеживается!

# Правильный порядок:
git rm --cached config.local.js   # удалить из отслеживания
echo "config.local.js" >> .gitignore  # добавить в игнор
git add .gitignore
git commit -m "Stop tracking config.local.js"
# Теперь изменения в config.local.js игнорируются
```

## git mv — связанная команда

Переименование файлов в Git похоже на удаление старого и добавление нового. Для этого есть специальная команда `git mv`:

```bash
# Переименовать файл
git mv old-name.js new-name.js

# Это эквивалентно:
mv old-name.js new-name.js
git rm old-name.js
git add new-name.js

# После git mv файл готов к коммиту
git commit -m "Rename old-name.js to new-name.js"
```

---

## Часто задаваемые вопросы

### В чём разница между rm и git rm?

`rm` удаляет файл только с диска. Git продолжает его отслеживать и покажет в статусе как «deleted». `git rm` удаляет файл с диска и одновременно регистрирует удаление в индексе — готово к коммиту.

### Как удалить файл из Git, не удаляя с диска?

Используй `git rm --cached имя-файла`. Файл останется локально, но Git перестанет его отслеживать. Обычно после этого нужно добавить файл в `.gitignore`.

### Что если я случайно сделал git rm?

До коммита можно восстановить: `git restore имя-файла` или `git checkout -- имя-файла`. После коммита — через `git restore --source=HEAD~1 имя-файла`.

### Нужно ли коммитить после git rm?

Да. `git rm` только регистрирует удаление в индексе. Чтобы изменение попало в историю репозитория, нужен коммит.

### Как удалить файл только из удалённого репозитория, не трогая локальный?

Сначала удали из локального репозитория (`git rm --cached`), закоммить, потом запуши. Файл пропадёт из удалённого репозитория при следующем `git push`.

---

## Читай также

- [Что такое коммит в программировании]({{< relref "chto-takoe-kommit" >}})
- [Что такое коммит в программировании]({{< relref "chto-takoe-kommit" >}})
- [Что такое репозиторий в Git]({{< relref "chto-takoe-repozitoriy" >}})
