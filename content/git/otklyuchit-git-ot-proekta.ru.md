---
title: "Как отключить Git от проекта: удаление папки .git"
description: "Инструкция по отключению Git от проекта. Удаление папки .git, отмена отслеживания файлов через git rm --cached, реинициализация Git."
date: 2026-02-21
lastmod: 2026-02-21
draft: false
slug: "otklyuchit-git-ot-proekta"
keywords: ["отключить git от проекта", "удалить git из папки", "удалить .git папку", "как удалить git из проекта", "удалить .git директорию", "отсоединить папку от git"]
tags: ["git", "beginner"]
categories: ["git"]
---

Иногда нужно полностью убрать Git из проекта: удалить историю, начать заново или просто перестать отслеживать папку как репозиторий. Это простая операция — Git хранит всё в одной скрытой папке `.git`.

## Что хранится в папке .git

Вся история коммитов, конфигурация, ветки и индекс Git находятся в папке `.git` в корне репозитория:

```
my-project/
├── .git/           ← здесь всё, что знает Git
│   ├── config      — конфигурация репозитория
│   ├── HEAD        — указатель на текущую ветку
│   ├── objects/    — все коммиты, файлы, теги
│   ├── refs/       — ветки и теги
│   └── index       — индекс (staged files)
├── src/
├── README.md
└── package.json
```

Удаление папки `.git` полностью отключает Git от проекта. Файлы проекта остаются нетронутыми.

## Способ 1: удалить .git папку полностью

Если нужно полностью убрать Git из проекта:

```bash
# Linux / macOS / Git Bash
rm -rf .git

# Windows (PowerShell)
Remove-Item -Recurse -Force .git

# Windows (Command Prompt)
rmdir /s /q .git
```

После этого:
- Папка больше не является Git репозиторием
- Все команды `git` перестанут работать в этой папке
- История коммитов, ветки, теги — всё удалено
- Сами файлы проекта остаются без изменений

Проверить, что Git удалён:

```bash
git status
# fatal: not a git repository (or any of the parent directories): .git
```

## Отключить от удалённого репозитория, сохранив локальный Git

Если нужно убрать связь с GitHub/GitLab, но оставить локальную историю:

```bash
# Посмотреть текущие remote
git remote -v
# origin  git@github.com:username/project.git (fetch)
# origin  git@github.com:username/project.git (push)

# Удалить remote origin
git remote remove origin

# Проверить — теперь нет удалённых репозиториев
git remote -v
```

Локальный Git работает, история сохранена, но `git push` и `git pull` теперь не с чем синхронизироваться.

## Способ 2: убрать файлы из отслеживания (git rm --cached)

Если нужно оставить Git, но перестать отслеживать конкретные файлы (например, добавить их в `.gitignore`):

```bash
# Убрать один файл из отслеживания
git rm --cached secrets.env

# Убрать папку из отслеживания
git rm --cached -r node_modules/

# Убрать все файлы из отслеживания (но сохранить на диске)
git rm --cached -r .

# Добавить в .gitignore
echo "secrets.env" >> .gitignore
echo "node_modules/" >> .gitignore

# Закоммитить изменения
git add .gitignore
git commit -m "Remove tracked files, update .gitignore"
```

`git rm --cached` убирает файл из индекса Git (перестаёт отслеживать), но оставляет файл на диске. После этого нужно добавить файл в `.gitignore`, чтобы он не добавился снова при `git add .`.

## Реинициализация Git с нуля

Если нужно удалить всю историю и начать заново:

```bash
# Удалить Git
rm -rf .git

# Инициализировать заново
git init

# Добавить все файлы
git add .

# Первый коммит с нуля
git commit -m "Initial commit"

# Если нужно подключить к тому же удалённому репозиторию
git remote add origin git@github.com:username/project.git

# Force push (перезапишет историю на GitHub!)
git push --force origin main
```

**Внимание**: force push на общий репозиторий удалит историю для всех участников. Убедитесь, что это нужно всей команде.

## Отключить Git для вложенного репозитория

Если внутри проекта оказался вложенный репозиторий (случайно сделали `git init` в подпапке):

```bash
# Найти вложенные .git папки
find . -name ".git" -type d

# Вывод:
# ./.git                    — основной репозиторий
# ./vendor/some-lib/.git    — вложенный (нежелательный)

# Удалить вложенный .git
rm -rf ./vendor/some-lib/.git

# Теперь можно добавить содержимое папки в основной репозиторий
git add vendor/some-lib/
git commit -m "Add vendor library as regular directory"
```

## Что происходит с файлами после удаления .git

Это важно понимать: удаление `.git` влияет только на историю и метаданные Git:

```bash
# До удаления .git
ls -la
# .git/
# src/
# README.md
# package.json

rm -rf .git

# После удаления .git
ls -la
# src/          ← остался
# README.md     ← остался
# package.json  ← остался
# (нет .git)
```

Все рабочие файлы сохранены. Git просто перестаёт существовать для этой папки.

## Практические примеры

```bash
# Случай 1: начать заново, удалив всю историю
rm -rf .git
git init
git add .
git commit -m "Fresh start"

# Случай 2: убрать связь с форком, сохранив историю
git remote remove origin
git remote add origin git@github.com:myaccount/newrepo.git
git push -u origin main

# Случай 3: перестать отслеживать .env файл
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "Stop tracking .env file"

# Случай 4: случайно инициализировали Git не в той папке
cd /wrong/folder
ls -la .git   # убедиться что это не нужный репозиторий
rm -rf .git
cd /correct/project
git status    # убедиться что нужный репозиторий работает

# Случай 5: проверить, является ли папка Git репозиторием
git rev-parse --is-inside-work-tree 2>/dev/null
# true  — является репозиторием
# (пустой вывод) — не является
```

## Удаление репозитория на GitHub (опционально)

Если также нужно удалить репозиторий на GitHub (это отдельная операция от локального удаления `.git`):

```
1. Открыть репозиторий на GitHub

2. Settings → Danger Zone → Delete this repository

3. Ввести имя репозитория для подтверждения

4. Нажать "I understand the consequences, delete this repository"
```

Удаление на GitHub не влияет на локальные файлы. И наоборот — удаление `.git` локально не удаляет репозиторий на GitHub.

## Часто задаваемые вопросы

**Удалятся ли мои файлы проекта, если удалить .git?** Нет. Удаляется только папка `.git` со всей историей Git. Файлы проекта (`src/`, `README.md`, и т. д.) остаются нетронутыми.

**Как проверить, что папка .git удалена корректно?** Выполните `git status` — должно появиться сообщение "fatal: not a git repository". Или `ls -la` — не должно быть папки `.git`.

**Можно ли восстановить историю после удаления .git?** Нет, если не было резервной копии. Папка `.git` содержит всю историю — после её удаления данные безвозвратно потеряны.

**Чем отличается `git rm --cached` от удаления .git?** `git rm --cached` убирает конкретный файл из отслеживания Git, но оставляет репозиторий и историю. Удаление `.git` полностью уничтожает весь репозиторий.

**Как убрать Git из проекта на Windows?** Папка `.git` скрыта. Через PowerShell: `Remove-Item -Recurse -Force .git`. Через проводник: включить показ скрытых файлов, найти `.git`, удалить.

## Заключение

Отключить Git от проекта просто: удалите папку `.git` командой `rm -rf .git`. Это безопасно для файлов проекта — они останутся. Если нужно только убрать конкретный файл из отслеживания, используйте `git rm --cached`. Для управления удалёнными репозиториями — `git remote remove origin`. После удаления `.git` можно инициализировать Git заново через `git init`.

## По теме

- [git init]({{< relref "git-init" >}})
- [Что такое репозиторий]({{< relref "chto-takoe-repozitoriy" >}})
- [git rm]({{< relref "git-rm" >}})
- [Отключение от remote]({{< relref "git-remote-add" >}})
