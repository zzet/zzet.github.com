---
title: "Объекты Git: blob, tree, commit и tag изнутри"
description: "Подробно об объектах Git: blob, tree, commit, tag. Хранение объектов, SHA-1 хеширование, pack-файлы, дедупликация данных. Как Git хранит историю проекта."
date: 2026-01-18
lastmod: 2026-01-18
draft: false
slug: "git-internals-objects"
keywords: ["git объекты", "git blob tree commit tag", "git внутренние объекты хранилище", "git blob что это", "git tree object", "git commit object структура"]
tags: ["git", "advanced"]
categories: ["git"]
---

Внутри Git всё хранится как объекты. Это не абстракция — буквально любые данные в репозитории: содержимое каждого файла, структура директорий, коммиты и теги — это объекты в контентно-адресуемом хранилище. Понимание объектной модели Git объясняет большинство особенностей его поведения.

## Git как хранилище объектов

Git можно представить как простую базу данных вида «ключ → значение». Ключ — SHA-1 хеш содержимого, значение — сам объект:

```bash
# Создать объект вручную (низкоуровневый способ)
echo "test content" | git hash-object --stdin -w
# d670460b4b4aece5915caf5c68d12f560a9fe3e4

# Прочитать его обратно по хешу
git cat-file -p d670460b4b
# test content

# Тип объекта
git cat-file -t d670460b4b
# blob
```

Объект создан. Он хранится в `.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4` — первые два символа хеша образуют имя директории, остальные 38 — имя файла.

## Blob: содержимое файлов

Blob (Binary Large Object) хранит содержимое файла целиком — без имени, без метаданных, только байты:

```bash
# Blob создаётся при git add
echo "# My Project" > README.md
git add README.md

# Посмотреть хеш blob в индексе
git ls-files --stage README.md
# 100644 8ec9a00bfd09b3190ac6b22251dbb1aa95a0579d 0	README.md

# Содержимое blob
git cat-file -p 8ec9a00b
# # My Project

# Размер blob
git cat-file -s 8ec9a00b
# 14
```

Важная особенность: **два файла с одинаковым содержимым — это один blob**. Git автоматически дедуплицирует:

```bash
# Создать два файла с одинаковым содержимым
echo "same content" > file1.txt
echo "same content" > file2.txt
git add file1.txt file2.txt

git ls-files --stage
# 100644 69a34e... 0	file1.txt
# 100644 69a34e... 0	file2.txt
# Одинаковый хеш! Один объект blob для двух файлов.
```

## Tree: структура директорий

Tree описывает директорию: список записей blob и tree с именами и правами:

```bash
# Посмотреть tree HEAD коммита
git cat-file -p HEAD^{tree}
```

```
100644 blob 8ec9a00b...  README.md
100644 blob f3b2c8a1...  package.json
040000 tree 9d2f1b4e...  src
040000 tree 5a8c3e2f...  tests
```

```bash
# Посмотреть вложенный tree
git cat-file -p 9d2f1b4e
# 100644 blob b1c2d3e4...  index.js
# 100644 blob c2d3e4f5...  utils.js
# 040000 tree d3e4f5a6...  components

# ls-tree показывает то же самое удобнее
git ls-tree HEAD
git ls-tree -r HEAD  # рекурсивно (все файлы)
git ls-tree -r --name-only HEAD  # только пути
```

Права в tree:

```
100644 — обычный файл
100755 — исполняемый файл (chmod +x)
120000 — символическая ссылка
040000 — директория (tree)
160000 — gitlink (ссылка на submodule)
```

## Commit: снимок истории

Commit связывает tree (снимок проекта) с метаданными и родительским коммитом:

```bash
git cat-file -p HEAD
```

```
tree 9d2f1b4e5a8c3e2f1b4e5a8c3e2f1b4e5a8c3e2f
parent a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
author Иван Иванов <ivan@example.com> 1710000000 +0300
committer Иван Иванов <ivan@example.com> 1710000000 +0300

feat: add user authentication module
```

Первый коммит репозитория не имеет строки `parent`. Merge commit имеет два `parent`:

```bash
# Merge commit
git cat-file -p <merge-commit-hash>
# tree ...
# parent <хеш-ветки-main>
# parent <хеш-ветки-feature>
# author ...
```

Разница между author и committer: author — тот кто написал изменения, committer — тот кто применил коммит (при `git cherry-pick`, `git rebase` или patch workflow они могут различаться).

## Tag: метка на объект

Аннотированный тег — отдельный объект, указывающий на другой объект (обычно commit):

```bash
git tag -a v1.0.0 -m "Release 1.0.0"
git cat-file -p v1.0.0
```

```
object a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
type commit
tag v1.0.0
tagger Иван Иванов <ivan@example.com> 1710000000 +0300

Release 1.0.0

Changelog:
- User authentication
- Bug fixes in login flow
```

Лёгкий тег (lightweight) — просто файл в `refs/tags/` с хешем без создания объекта:

```bash
git tag v1.0.0-light  # лёгкий тег

git cat-file -t v1.0.0       # tag
git cat-file -t v1.0.0-light # commit ← напрямую на коммит
```

## Граф объектов

Связи между объектами образуют направленный ациклический граф (DAG):

```
HEAD → refs/heads/main → commit C3
                              ↓
                         tree T3
                        /    |    \
                    blob B1  blob B2  tree T2
                  (README) (pkg.json)  (src/)
                                        |
                                     blob B3
                                   (index.js)

commit C3 → commit C2 → commit C1 (root)
```

Каждый коммит ссылается на полный tree — **снимок всего проекта в этот момент**. Не дельту изменений, а полный снимок. Неизменённые файлы повторно используют те же blob объекты.

## Физическое хранение

### Свободные объекты (loose objects)

Каждый объект хранится в отдельном файле в `.git/objects/`:

```bash
# Хеш: d670460b4b4aece5915caf5c68d12f560a9fe3e4
# Файл: .git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4

# Файл сжат zlib (не читается как текст напрямую)
file .git/objects/d6/70460b...
# .../d6/70...: zlib compressed data

# Распаковать вручную (Python)
python3 -c "import zlib,sys; print(zlib.decompress(open(sys.argv[1],'rb').read()))" \
  .git/objects/d6/70460b...
# b'blob 13\x00Hello, Git!\n'
```

### Pack-файлы

После накопления объектов Git упаковывает их в pack-файл с дельта-сжатием:

```bash
# Принудительно упаковать
git gc
# Counting objects: 124, done.
# Delta compression using up to 8 threads.
# Compressing objects: 100% (89/89), done.

# Pack-файлы в objects/pack/
ls .git/objects/pack/
# pack-abc123.idx   ← индекс для быстрого поиска
# pack-abc123.pack  ← сами данные

# Список объектов в pack-файле
git verify-pack -v .git/objects/pack/pack-abc123.idx | head -10
```

### Дедупликация через хеши

Поскольку хеш объекта зависит только от его содержимого, Git автоматически избегает дублирования:

```bash
# Сделать 100 коммитов без изменения большого файла
# Blob для этого файла создаётся ОДИН РАЗ
git count-objects -v
# count: 0
# size: 0
# in-pack: 42    ← все в pack-файле
# packs: 1
# size-pack: 168 ← килобайт
```

## Вычисление хешей

SHA-1 вычисляется из типа, размера и содержимого объекта:

```bash
# Формат: "<тип> <размер>\0<содержимое>"

# Для blob:
printf "blob 13\0Hello, Git!\n" | sha1sum
# d670460b4b4aece5915caf5c68d12f560a9fe3e4

# Git даёт тот же результат
echo "Hello, Git!" | git hash-object --stdin
# d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

Для tree и commit формат сложнее, но принцип тот же — детерминированный хеш от содержимого.

## Проверка целостности

```bash
# Проверить все объекты
git fsck
# Checking object connectivity and validity.
# (пустой вывод = всё хорошо)

# С подробностями
git fsck --full --strict

# Найти объекты, недоступные ни из одной ссылки
git fsck --unreachable

# Количество объектов
git count-objects -v
```

## Часто задаваемые вопросы

**Почему Git хранит полные снимки, а не разницы?** Полные снимки позволяют эффективно переключаться между ветками — не нужно применять серию патчей. Дедупликация через хеши компенсирует размер. Pack-файлы добавляют дельта-сжатие для экономии места.

**Что происходит с объектами при удалении ветки?** Объекты остаются в `.git/objects/` как «мусор». `git gc` удаляет объекты, недостижимые из любой ссылки и старше grace period (14 дней по умолчанию).

**Как Git находит объект по хешу?** Для свободных объектов — просто открывает файл по пути. Для pack-файлов — использует .idx индекс для бинарного поиска.

**Можно ли доверять хешам?** SHA-1 теоретически уязвим к collision attacks. Практически это не является проблемой для большинства репозиториев. Git переходит на SHA-256 (`git init --object-format=sha256`).

**Что такое объекты-«сироты»?** Объекты, недоступные ни из одной ссылки (ветки, теги, HEAD). Появляются при `git reset`, `git rebase`, `git commit --amend`. Удаляются через `git gc`.

## Заключение

Git хранит все данные как объекты четырёх типов в контентно-адресуемом хранилище: blob (файлы), tree (директории), commit (снимки проекта), tag (метки). Объекты идентифицируются SHA-1 хешем, что обеспечивает дедупликацию и целостность. Pack-файлы эффективно сжимают данные. Для изучения объектов используйте [git cat-file]({{< relref "git-cat-file" >}}).

## По теме
- [Символические ссылки в Git (blob 120000)]({{< relref "git-symlinks" >}})

- [Git refs: ссылки изнутри]({{< relref "git-refs" >}})
