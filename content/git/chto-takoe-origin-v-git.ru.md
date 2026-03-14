---
title: "Что такое origin в Git: полное объяснение"
description: "Узнайте что означает origin в Git. Удалённый репозиторий, origin/master, git push/pull, примеры команд и практические советы."
date: 2026-01-02
lastmod: 2026-01-02
draft: false
slug: "chto-takoe-origin-v-git"
keywords:
  - что значит origin в git
  - git origin что это
  - origin git это
  - git origin
  - git origin master
tags:
  - git
  - beginner
categories:
  - Git
aliases: []
---

Если ты только начинаешь работать с Git, ты наверняка видел команды вроде `git push origin main` или `git pull origin develop` и задавался вопросом: что такое этот `origin`? Короткий ответ: origin — это просто имя, псевдоним для URL удалённого репозитория. Не часть Git как такового, а соглашение об именовании. В этой статье разберём подробно: откуда берётся origin, что он означает и как с ним работать.

## Что такое origin

Origin — это имя (алиас) для удалённого репозитория. По умолчанию, когда ты клонируешь репозиторий, Git автоматически называет источник клонирования `origin`.

Вместо того чтобы каждый раз писать полный URL:

```bash
# Без origin — неудобно
git push https://github.com/username/my-project.git main

# С origin — удобно
git push origin main
```

`origin` — это просто сокращение для URL. Технически ты можешь назвать его как угодно: `upstream`, `github`, `production` — Git не требует конкретного имени.

## Откуда берётся origin

Origin появляется автоматически при клонировании:

```bash
git clone https://github.com/username/project.git
cd project

# Git автоматически создал origin, указывающий на URL откуда клонировали
git remote -v
# origin  https://github.com/username/project.git (fetch)
# origin  https://github.com/username/project.git (push)
```

Если ты создал репозиторий локально через `git init`, origin нужно добавить вручную:

```bash
git init
git remote add origin https://github.com/username/project.git

# Проверить
git remote -v
# origin  https://github.com/username/project.git (fetch)
# origin  https://github.com/username/project.git (push)
```

## origin vs другие удалённые репозитории

У одного локального репозитория может быть несколько удалённых. Классический пример — работа с форком open source проекта:

```bash
# origin — твой форк (куда ты пушишь)
git remote add origin https://github.com/YOU/project.git

# upstream — оригинальный репозиторий (откуда забираешь обновления)
git remote add upstream https://github.com/ORIGINAL/project.git

# Посмотреть все удалённые репозитории
git remote -v
# origin    https://github.com/YOU/project.git (fetch)
# origin    https://github.com/YOU/project.git (push)
# upstream  https://github.com/ORIGINAL/project.git (fetch)
# upstream  https://github.com/ORIGINAL/project.git (push)
```

Теперь ты можешь забирать обновления из оригинального проекта:

```bash
git fetch upstream
git merge upstream/main
```

И отправлять свои изменения в свой форк:

```bash
git push origin my-feature
```

## origin/master и origin/main — что это такое

Когда ты видишь `origin/main` или `origin/master` — это не локальная ветка. Это **удалённая ветка** — локальная копия ветки из удалённого репозитория. Git хранит их отдельно от твоих локальных веток.

```bash
# Посмотреть все ветки: локальные и удалённые
git branch -a
# * main                     ← локальная ветка (ты здесь)
#   feature/auth             ← локальная ветка
#   remotes/origin/main      ← удалённая ветка
#   remotes/origin/develop   ← удалённая ветка

# Посмотреть только удалённые ветки
git branch -r
# origin/main
# origin/develop
```

`origin/main` обновляется когда ты делаешь `git fetch origin` — Git скачивает последнее состояние удалённой ветки. Это не то же самое, что `git pull` — fetch только обновляет твоё знание о состоянии сервера, не сливая изменения в локальную ветку.

```bash
# Обновить информацию об удалённых ветках без слияния
git fetch origin

# Сравнить свою ветку с удалённой
git diff main origin/main

# Посмотреть коммиты которые есть на сервере но не у тебя
git log main..origin/main

# Посмотреть коммиты которые есть у тебя но не на сервере
git log origin/main..main
```

## Просмотр информации об origin

```bash
# Проверить URL origin
git remote -v
# origin  https://github.com/username/project.git (fetch)
# origin  https://github.com/username/project.git (push)

# Получить конкретный URL
git config --get remote.origin.url
# https://github.com/username/project.git

# Подробная информация об origin
git remote show origin
# * remote origin
#   Fetch URL: https://github.com/username/project.git
#   Push  URL: https://github.com/username/project.git
#   HEAD branch: main
#   Remote branches:
#     develop tracked
#     main    tracked
#   Local branch configured for 'git pull':
#     main merges with remote main
#   Local ref configured for 'git push':
#     main pushes to main (up to date)
```

## Работа с origin: push и pull

Основные операции:

```bash
# Отправить локальную ветку main в origin
git push origin main

# Получить изменения из origin и слить в текущую ветку
git pull origin main

# Только скачать изменения без слияния
git fetch origin

# При первом пуше новой ветки — установить связь (upstream)
git push -u origin feature/auth
# После этого можно просто: git push (без указания origin и ветки)

# Удалить ветку на сервере
git push origin --delete feature/old-auth
```

Флаг `-u` (или `--set-upstream`) при первом `push` устанавливает связь между локальной и удалённой веткой. После этого Git знает куда пушить и откуда пулить по умолчанию:

```bash
# Первый раз с -u
git push -u origin feature/auth

# Все последующие разы — просто
git push
git pull
```

## Изменение URL origin

Если репозиторий переехал или ты хочешь переключиться с HTTPS на SSH:

```bash
# Посмотреть текущий URL
git remote -v

# Изменить URL origin
git remote set-url origin git@github.com:username/project.git

# Проверить
git remote -v
# origin  git@github.com:username/project.git (fetch)
# origin  git@github.com:username/project.git (push)
```

## Управление удалёнными репозиториями

```bash
# Добавить новый удалённый репозиторий
git remote add backup https://gitlab.com/username/project.git

# Переименовать удалённый репозиторий
git remote rename origin github

# Удалить удалённый репозиторий (только запись в конфиге, не сам репозиторий)
git remote remove backup

# Обновить список удалённых веток (удалить удалённые ветки которых нет на сервере)
git fetch --prune origin
# или сокращённо
git fetch -p
```

## Клонирование с другим именем для origin

Если хочешь изначально назвать remote иначе, а не `origin`:

```bash
# Клонировать и назвать remote 'github' вместо 'origin'
git clone -o github https://github.com/username/project.git

git remote -v
# github  https://github.com/username/project.git (fetch)
# github  https://github.com/username/project.git (push)
```

---

## Часто задаваемые вопросы

### Почему origin называется именно "origin"?

Это историческое соглашение. Origin означает «источник» — репозиторий, из которого пришёл код. Git автоматически использует это имя при `git clone`. Ты можешь переименовать origin как угодно, но менять соглашение без причины не стоит — это запутает коллег.

### Что означает "origin/master" или "origin/main"?

Это удалённая ветка — локальная копия ветки `master` (или `main`) из репозитория `origin`. Git обновляет её при `git fetch`. Это не то же самое, что твоя локальная ветка `master` или `main`.

### Могу ли я изменить origin на другое имя?

Да: `git remote rename origin <новое-имя>`. Только помни, что все команды теперь нужно обновить: вместо `git push origin main` будет `git push <новое-имя> main`.

### Что произойдёт если я удалю origin?

Git удалит запись об этом удалённом репозитории из конфигурации. Сам удалённый репозиторий (на GitHub и т.д.) останется нетронутым. Ты потеряешь только связь с ним — которую можно восстановить через `git remote add origin <url>`.

### Может ли быть несколько origin?

Нет, у каждого имени должен быть один URL. Но можно добавить несколько URL для одного имени через `git remote set-url --add`. Это позволяет пушить в несколько репозиториев одной командой. Более распространённый подход — использовать разные имена: `origin`, `upstream`, `backup`.

---

## Читай также

- [Что такое репозиторий в Git]({{< relref "chto-takoe-repozitoriy" >}})
- [git init: как создать репозиторий]({{< relref "git-init" >}})
- [Удалённые репозитории и Git-сервер]({{< relref "2014-03-28-lection-4-git-course-undev" >}})
