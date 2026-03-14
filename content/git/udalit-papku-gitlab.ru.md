---
title: "Как удалить папку в GitLab: через веб-интерфейс и git rm"
description: "Инструкция по удалению папки в GitLab. Команда git rm -r, удаление через веб-интерфейс GitLab, удаление из истории через BFG."
date: 2026-03-10
lastmod: 2026-03-10
draft: false
slug: "udalit-papku-gitlab"
keywords: ["удалить папку gitlab", "как удалить директорию gitlab", "git rm папка", "gitlab удалить папку браузер", "git rm -r директория", "удалить каталог gitlab"]
tags: ["git", "gitlab", "beginner"]
categories: ["git"]
---

Удалить папку из GitLab репозитория можно двумя способами: через командную строку с `git rm -r` или через веб-интерфейс GitLab. В обоих случаях папка исчезнет из текущей версии, но останется в истории коммитов — это нормальное поведение Git.

## Способ 1: удаление через команду git rm

Самый надёжный способ — через командную строку:

```bash
# Клонировать репозиторий (если ещё нет локально)
git clone git@gitlab.com:username/project.git
cd project

# Посмотреть содержимое (убедиться, что папка существует)
ls -la

# Удалить папку рекурсивно
git rm -r old-folder/

# Вывод:
# rm 'old-folder/file1.txt'
# rm 'old-folder/subdir/file2.txt'
# rm 'old-folder/subdir/file3.js'

# Проверить статус
git status
# Changes to be committed:
#   deleted:    old-folder/file1.txt
#   deleted:    old-folder/subdir/file2.txt

# Создать коммит
git commit -m "Remove old-folder directory"

# Отправить на GitLab
git push origin main
```

## Сухой прогон перед удалением

Если не уверены, что именно будет удалено — используйте флаг `--dry-run`:

```bash
# Посмотреть, что будет удалено, без реального удаления
git rm -r --dry-run old-folder/

# Вывод:
# rm 'old-folder/README.md'
# rm 'old-folder/config.json'
# rm 'old-folder/src/main.js'
# (реального удаления не произошло)
```

После проверки — повторить без `--dry-run`.

## Разница между git rm и просто удалением папки

```bash
# Обычное удаление (файлы удалены с диска, Git не знает)
rm -rf old-folder/

# Нужно явно сообщить Git об удалении
git add -u
# или
git add old-folder/  # для удалённых файлов

# git rm делает всё за один шаг:
git rm -r old-folder/
# = удаляет папку С диска И регистрирует удаление в индексе
```

Рекомендуется использовать `git rm -r` — меньше шагов и нет риска забыть.

## Удалить папку из индекса, но оставить на диске

Если папка случайно добавлена в репозиторий и её нужно исключить (например, `node_modules`, `build`, `.env`):

```bash
# Убрать из Git контроля, не удаляя с диска
git rm -r --cached node_modules/

# Добавить в .gitignore чтобы не добавлялась снова
echo "node_modules/" >> .gitignore

# Закоммитить
git add .gitignore
git commit -m "Remove node_modules from tracking, add to .gitignore"
git push origin main
```

После этого `node_modules/` останется на диске локально, но не будет отслеживаться Git.

## Способ 2: удаление через веб-интерфейс GitLab

GitLab позволяет удалять папки через браузер:

```
1. Открыть репозиторий на GitLab
2. Перейти в нужную папку (нажать на неё в списке файлов)
3. Нажать кнопку с тремя точками (...) или иконку действий
   рядом с названием папки
4. Выбрать "Delete directory"
5. В диалоге подтвердить:
   - Commit message (например: "Remove old-folder")
   - Target branch (ветка для коммита)
6. Нажать "Delete directory"
```

Если кнопка удаления не видна — убедитесь, что у вас есть права Developer или Maintainer в проекте.

## Удаление через GitLab Web IDE

Альтернативный способ через встроенный редактор:

```
1. Открыть репозиторий
2. Нажать кнопку "Web IDE" (или Edit → Web IDE)
3. В дереве файлов слева найти папку
4. Правый клик → Delete
5. Подтвердить удаление
6. В Source Control (иконка слева) написать сообщение коммита
7. Нажать "Commit to main"
```

Web IDE позволяет удалить несколько файлов/папок в одном коммите — это удобнее, чем стандартный веб-редактор.

## Удаление папки из всей истории репозитория

Если папка содержала секреты или конфиденциальные данные и её нужно удалить из всей истории:

```bash
# Инструмент BFG Repo-Cleaner
# Скачать: https://rtyley.github.io/bfg-repo-cleaner/

# Клонировать репозиторий как bare (зеркало)
git clone --mirror git@gitlab.com:username/project.git

# Удалить папку из всей истории
java -jar bfg.jar --delete-folders old-folder project.git

# Очистить рефлоги
cd project.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Force push
git push --force
```

**Важно**: после force push все участники команды должны заново клонировать репозиторий — их локальные копии устарели.

Альтернатива BFG — `git filter-repo`:

```bash
# Установить git-filter-repo
pip install git-filter-repo

# Удалить папку из всей истории
git filter-repo --path old-folder/ --invert-paths

# Force push
git push origin --force --all
```

## Восстановление удалённой папки

Если ещё не закоммитили:

```bash
# Восстановить папку из последнего коммита
git restore old-folder/
```

Если уже создали коммит (но не push):

```bash
# Отменить последний коммит, сохранив изменения
git reset --soft HEAD~1
# Восстановить папку
git restore old-folder/
```

Если уже запушили на GitLab:

```bash
# Создать revert-коммит
git revert HEAD
git push origin main
```

## Практические примеры

```bash
# Удалить папку с логами
git rm -r logs/
git commit -m "Remove logs directory"
git push origin main

# Удалить папку и исключить из отслеживания
git rm -r --cached dist/
echo "dist/" >> .gitignore
git add .gitignore
git commit -m "Ignore dist folder"

# Удалить несколько папок одним коммитом
git rm -r old-feature/ deprecated-api/ temp/
git commit -m "Remove deprecated code directories"
git push origin main

# Сухой прогон для проверки
git rm -r --dry-run src/legacy/

# Удалить папки по маске
git rm -r **/__pycache__/
git commit -m "Remove Python cache directories"
```

## Часто задаваемые вопросы

**Останется ли папка в истории коммитов после удаления?** Да. `git rm -r` создаёт новый коммит с удалением. Папка доступна в старых коммитах через `git show <hash>:path/to/folder`. Для удаления из всей истории — BFG или git filter-repo.

**Как удалить пустую папку?** Git не отслеживает пустые папки. Если папка пустая — она не в репозитории. Обычно в пустых папках оставляют файл `.gitkeep` чтобы Git отслеживал папку.

**Удалятся ли локальные файлы при `git rm -r`?** Да, файлы удаляются с диска. Если нужно только убрать из Git — используйте `git rm -r --cached`.

**Почему в веб-интерфейсе GitLab нет кнопки удаления папки?** Минимально необходимая роль — Developer. Если вы Guest или Reporter — кнопки удаления нет. Также кнопка может быть скрыта, если ветка защищена.

**Как удалить папку во всех ветках?** `git rm -r` удаляет только в текущей ветке. Для каждой ветки нужно повторить операцию, или использовать BFG/git filter-repo для удаления из всей истории всех веток.

## Заключение

Удалить папку в GitLab репозитории: `git rm -r folder/`, затем `git commit` и `git push`. Через веб-интерфейс — перейти в папку, три точки, "Delete directory". Для удаления только из отслеживания (сохранив файлы) — `git rm -r --cached folder/`. Для удаления из всей истории (секреты, пароли) — BFG Repo-Cleaner. Управление ролями для доступа к удалению — [роли GitLab]({{< relref "gitlab-reporter" >}}).
