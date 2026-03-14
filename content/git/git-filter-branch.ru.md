---
title: "git filter-branch: переписывание истории и удаление файлов"
description: "Руководство по git filter-branch. Удаление файлов из всей истории, изменение email автора, современная альтернатива git filter-repo."
date: 2026-01-12
lastmod: 2026-01-12
draft: false
slug: "git-filter-branch"
keywords: ["git filter branch", "удаление файлов из истории git", "переписать историю git", "git filter-branch примеры", "удалить файл из всей истории git", "git filter-repo"]
tags: ["git", "advanced"]
categories: ["git"]
---

`git filter-branch` — инструмент для переписывания истории репозитория. Позволяет удалить файл из каждого коммита, изменить email автора, трансформировать дерево файлов. Команда мощная, но опасная и медленная. Современная альтернатива — `git filter-repo`.

## Когда используется filter-branch

Типичные сценарии:

```
- Случайно закоммитили пароли или API ключи
- Нужно удалить большой файл из всей истории
- Изменить email/имя автора во всех коммитах
- Вычленить подпапку в отдельный репозиторий
- Объединить несколько репозиториев
```

**Важно**: после filter-branch все SHA-хеши коммитов изменятся. Все, у кого есть клон репозитория, должны будут заново клонировать.

## Удаление файла из всей истории

Наиболее частый сценарий — удаление файла с секретами:

```bash
# Клонировать репозиторий (работать с копией!)
git clone git@github.com:username/project.git
cd project

# Удалить файл из каждого коммита
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secrets.env' \
  --prune-empty --tag-name-filter cat -- --all

# --force              — перезаписать существующие refs
# --index-filter       — фильтр для индекса (быстрее tree-filter)
# git rm --cached      — удалить из индекса
# --ignore-unmatch     — не ошибаться если файла нет в коммите
# --prune-empty        — удалить коммиты ставшие пустыми
# --tag-name-filter cat — обновить теги
# -- --all             — обработать все ветки

# Очистить refs и собрать мусор
git for-each-ref --format="delete %(refname)" refs/original | git update-ref --stdin
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Force push
git push origin --force --all
git push origin --force --tags
```

## Удаление папки из истории

```bash
# Удалить папку node_modules из всей истории
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch -r node_modules/' \
  --prune-empty --tag-name-filter cat -- --all

# После очистки
git gc --prune=now --aggressive
```

## Изменение email автора во всех коммитах

```bash
git filter-branch --env-filter '
OLD_EMAIL="old@example.com"
NEW_NAME="New Name"
NEW_EMAIL="new@example.com"

if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$NEW_NAME"
    export GIT_COMMITTER_EMAIL="$NEW_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$NEW_NAME"
    export GIT_AUTHOR_EMAIL="$NEW_EMAIL"
fi
' --tag-name-filter cat -- --all
```

## --tree-filter vs --index-filter

```bash
# --tree-filter: работает с файлами на диске (медленнее, но проще)
git filter-branch --tree-filter 'rm -f secrets.env' -- --all

# --index-filter: работает с индексом Git (быстрее)
git filter-branch --index-filter \
  'git rm --cached --ignore-unmatch secrets.env' -- --all

# Для больших репозиториев: --index-filter намного быстрее
```

## Вычленение подпапки в отдельный репозиторий

```bash
# Оставить только содержимое папки src/
git filter-branch --subdirectory-filter src/ -- --all

# Теперь содержимое src/ стало корнем репозитория
# Историю файлов вне src/ нет
```

## Современная альтернатива: git filter-repo

`git filter-branch` официально рекомендован к замене на `git filter-repo`. Причины:

```
- В 10-50 раз быстрее
- Безопаснее (не оставляет резервных refs)
- Более удобный синтаксис
- Активно поддерживается
```

```bash
# Установить git-filter-repo
pip install git-filter-repo
# или
brew install git-filter-repo

# Удалить файл из всей истории
git filter-repo --path secrets.env --invert-paths

# Удалить папку
git filter-repo --path node_modules/ --invert-paths

# Изменить email автора
git filter-repo --email-callback '
    return email if email != b"old@example.com" else b"new@example.com"
'

# Вычленить подпапку
git filter-repo --subdirectory-filter src/

# Force push (после filter-repo нужен fresh clone или --force)
git push origin --force --all
```

## После переписывания истории

Когда история переписана, нужно уведомить команду:

```bash
# 1. Force push на все ветки
git push origin --force --all
git push origin --force --tags

# 2. Инструкции для других разработчиков:
# Все локальные клоны устарели после force push
# Каждый разработчик должен:
git fetch origin
# Затем для каждой ветки:
git checkout main
git reset --hard origin/main
# ИЛИ проще:
git clone <repo-url> fresh-clone
```

## Отмена filter-branch

Если filter-branch создал refs/original:

```bash
# Вернуться к исходному состоянию (если ещё не gc)
git reset --hard refs/original/refs/heads/main

# Удалить резервные refs (после подтверждения результата)
git for-each-ref --format="delete %(refname)" refs/original | \
  git update-ref --stdin
```

## Проверка результата

```bash
# Убедиться что файл удалён из истории
git log --all --full-history -- secrets.env
# (должен быть пустой вывод)

# Проверить что файл не остался в каком-либо коммите
git grep 'password' $(git log --all --oneline | awk '{print $1}')
```

## Часто задаваемые вопросы

**Нужно ли также инвалидировать скомпрометированные ключи?** Да, обязательно. Удаление файла из истории Git не означает, что секрет в безопасности — он мог быть уже скопирован кем-то из просмотра истории. Смените все скомпрометированные ключи/пароли немедленно.

**Можно ли отменить filter-branch после force push?** Нет, если у других нет копии старой истории. Именно поэтому важно работать с клоном и убедиться в результате перед force push.

**Почему нужен git gc после filter-branch?** Git сохраняет старые объекты как мусор. Без `git gc --prune=now` старый файл останется в `.git/objects` и технически ещё доступен.

**Что такое --prune-empty?** После удаления файлов некоторые коммиты могут остаться пустыми (без изменений). `--prune-empty` удаляет такие коммиты из истории.

**git filter-repo или BFG?** Оба быстрее и удобнее filter-branch. `git filter-repo` предпочтительнее для большинства задач — активно разрабатывается. BFG проще для базовых сценариев (удалить файл, очистить большие файлы).

## Заключение

`git filter-branch` — мощный но устаревший инструмент для переписывания истории. Для новых проектов используйте `git filter-repo` (быстрее, проще, безопаснее). Основные сценарии: удаление секретов (`--path file --invert-paths`), изменение авторов (`--email-callback`), вычленение подпапки (`--subdirectory-filter`). После любого переписывания истории — смените скомпрометированные ключи и уведомьте команду о необходимости пересинхронизации. Подробнее об истории коммитов — [git reflog]({{< relref "git-reflog" >}}).
