---
title: "git cat-file: просмотр внутренних объектов Git"
description: "Команда git cat-file для просмотра объектов Git. Опции -t (тип), -s (размер), -p (содержимое). Работа с blob, tree, commit и tag объектами."
date: 2026-01-06
lastmod: 2026-01-06
draft: false
slug: "git-cat-file"
keywords: ["git cat-file", "просмотр объектов git", "git внутренние объекты команда", "git cat-file -t", "git cat-file -p", "тип объекта git"]
tags: ["git", "advanced"]
categories: ["git"]
---

`git cat-file` — низкоуровневая команда Git для просмотра содержимого, типа и размера объектов в хранилище объектов. Название происходит от Unix команды `cat` и «file» — файл объекта. Это основной инструмент для изучения внутреннего устройства Git.

## Основные опции

```bash
git cat-file [опция] <объект>
```

Три главные опции:

```bash
# -t: показать тип объекта
git cat-file -t <хеш>
# blob / tree / commit / tag

# -s: показать размер объекта в байтах
git cat-file -s <хеш>
# 1234

# -p: показать содержимое объекта (pretty-print)
git cat-file -p <хеш>
```

## Работа с blob объектами

Blob хранит содержимое файла:

```bash
# Найти хеш blob для файла в HEAD
git ls-tree HEAD README.md
# 100644 blob a8c4e7d3f2b1...  README.md

# Посмотреть содержимое
git cat-file -p a8c4e7d3
# # My Project
# This is the README file.

# Узнать тип
git cat-file -t a8c4e7d3
# blob

# Узнать размер
git cat-file -s a8c4e7d3
# 38
```

Получить хеш blob для конкретного файла в конкретном коммите:

```bash
# Хеш blob файла src/index.js в HEAD
git rev-parse HEAD:src/index.js

# Хеш blob в конкретном коммите
git rev-parse abc1234:src/index.js
```

## Работа с tree объектами

Tree представляет директорию:

```bash
# Посмотреть tree HEAD коммита
git cat-file -p HEAD^{tree}
```

```
100644 blob a8c4e7d3...  README.md
100644 blob f3b2c8a1...  package.json
040000 tree 9d2f1b4e...  src
040000 tree 5a8c3e2f...  tests
```

```bash
# Права файлов:
# 100644 — обычный файл
# 100755 — исполняемый файл
# 120000 — символическая ссылка
# 040000 — директория (tree)
# 160000 — gitlink (submodule)

# Посмотреть вложенный tree
git cat-file -p 9d2f1b4e
# 100644 blob b1c2d3e4...  index.js
# 100644 blob c2d3e4f5...  utils.js
```

## Работа с commit объектами

```bash
# Посмотреть последний коммит
git cat-file -p HEAD
```

```
tree 9d2f1b4e5a8c3e2f1b4e5a8c3e2f1b4e5a8c3e2f
parent a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
author Иван Иванов <ivan@example.com> 1710000000 +0300
committer Иван Иванов <ivan@example.com> 1710000000 +0300

feat: add user authentication
```

```bash
# Посмотреть конкретный коммит по хешу
git cat-file -p a1b2c3d4

# Посмотреть коммит по ветке
git cat-file -p main

# Посмотреть родительский коммит
git cat-file -p HEAD~1

# Merge commit имеет два родителя:
# parent a1b2c3d4...
# parent e5f6a7b8...
```

## Работа с tag объектами

Аннотированный тег — отдельный объект:

```bash
# Посмотреть тег
git cat-file -p v1.0.0
```

```
object a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
type commit
tag v1.0.0
tagger Иван Иванов <ivan@example.com> 1710000000 +0300

Release version 1.0.0

This release includes:
- User authentication
- Bug fixes
```

```bash
# Лёгкий тег (lightweight) ссылается напрямую на коммит:
git cat-file -t v1.0.0-light
# commit  ← не tag!
```

## Пакетный режим

Для обработки множества объектов:

```bash
# --batch: читает хеши из stdin, выводит метаданные и содержимое
echo "HEAD" | git cat-file --batch
# a1b2c3d4... commit 234
# tree 9d2f...
# parent a0b1...
# ...

# --batch-check: только метаданные (без содержимого)
echo "HEAD" | git cat-file --batch-check
# a1b2c3d4... commit 234

# Обработать все объекты репозитория
git cat-file --batch-check --batch-all-objects
# a1b2c3d4... commit 234
# b2c3d4e5... blob 38
# ...

# Статистика по типам объектов
git cat-file --batch-check --batch-all-objects | awk '{print $2}' | sort | uniq -c
#  42 blob
#  15 commit
#  15 tree
#   3 tag
```

## Практические примеры

```bash
# Извлечь содержимое файла из прошлого коммита
git cat-file -p HEAD~5:src/config.js > old-config.js

# Проверить, одинаковы ли два файла в разных коммитах
hash1=$(git rev-parse HEAD:file.txt)
hash2=$(git rev-parse HEAD~1:file.txt)
[ "$hash1" = "$hash2" ] && echo "Файлы одинаковы" || echo "Файлы различаются"

# Найти все большие blobs в репозитории
git cat-file --batch-check --batch-all-objects |
  awk '$2=="blob" {print $3, $1}' |
  sort -rn |
  head -10 |
  while read size hash; do
    echo "$size bytes: $(git log --all --oneline --find-object=$hash | head -1)"
  done

# Просмотреть полный граф коммитов вручную
git cat-file -p HEAD
git cat-file -p <parent-hash>  # перейти к родителю
```

## Ссылки на объекты

`git cat-file` принимает различные форматы ссылок:

```bash
# По полному SHA-1 хешу (40 символов)
git cat-file -p a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0

# По сокращённому хешу (минимум 4 символа)
git cat-file -p a1b2

# По имени ветки
git cat-file -p main

# По тегу
git cat-file -p v1.0.0

# По HEAD с указателями
git cat-file -p HEAD        # текущий коммит
git cat-file -p HEAD^       # родитель
git cat-file -p HEAD~3      # 3 коммита назад
git cat-file -p HEAD^{tree} # tree коммита HEAD
git cat-file -p HEAD:path   # объект по пути в коммите HEAD
```

## Разница с git show

`git show` — высокоуровневая команда с форматированием. `git cat-file` — низкоуровневая с сырыми данными:

```bash
# git show форматирует вывод для человека
git show HEAD
# commit a1b2c3d4... (HEAD -> main)
# Author: Иван Иванов <ivan@example.com>
# Date:   Fri Mar 13 2026
#
#     feat: add user authentication
#
# diff --git a/src/auth.js b/src/auth.js
# ...

# git cat-file показывает сырой объект
git cat-file -p HEAD
# tree 9d2f...
# parent ...
# author ...
# committer ...
#
# feat: add user authentication
```

## Часто задаваемые вопросы

**Зачем нужен git cat-file если есть git show и git log?** `git cat-file` работает напрямую с объектным хранилищем Git без интерпретации. Это полезно для скриптов, отладки и изучения внутреннего устройства Git.

**Как посмотреть все файлы, изменённые в коммите?** Через `git diff-tree --no-commit-id -r <хеш>` или `git show --stat <хеш>`. `git cat-file` для этого не предназначен.

**Можно ли создать объект через git cat-file?** Нет, `git cat-file` только читает. Для создания объектов используйте `git hash-object -w`.

**Что означает `missing` в выводе --batch-check?** Объект с указанным хешем не существует в репозитории.

**Как найти, в каком коммите появился файл с определённым хешем?** `git log --all --find-object=<хеш>` покажет все коммиты, которые касались этого объекта.

## Заключение

`git cat-file` — инструмент для прямого доступа к объектному хранилищу Git. Три основные опции: `-t` (тип), `-s` (размер), `-p` (содержимое). Пакетный режим `--batch` эффективен для массовой обработки. Команда полезна для изучения [внутреннего устройства Git]({{< relref "kak-ustroen-git-iznutri" >}}), отладки и написания скриптов.
