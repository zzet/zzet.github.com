---
title: "Как посмотреть удалённые ветки в Git: полное руководство"
description: "Как посмотреть удалённые ветки в Git. git branch -a, git fetch, отслеживание удалённых веток, командная разработка."
date: 2026-03-08
lastmod: 2026-03-08
draft: false
slug: "udalennye-vetki-git"
keywords: ["как посмотреть удаленные ветки в git", "удалённые ветки git", "remote branches", "git remote branches", "git branch -r", "git показать все ветки", "удалённые ветки команда git"]
tags: ["git", "branch", "remote"]
categories: ["git"]
---

При работе в команде ветки создаются не только локально, но и на сервере (GitHub, GitLab и т.д.). Чтобы видеть ветки коллег и работать с ними, нужно понять концепцию удалённых веток и знать команды для их просмотра.

## Что такое удалённые ветки

Удалённые ветки — это ветки на сервере (origin, upstream или другом удалённом репозитории). В локальном Git они представлены как `origin/main`, `origin/feature/auth` и т.д.

```
Удалённый сервер (GitHub):
  main
  feature/auth
  feature/cart

Локально у вас:
  main            ← ваша локальная ветка main
  feature/auth    ← ваша локальная ветка feature/auth
  origin/main     ← локальная копия серверной ветки main
  origin/feature/auth   ← локальная копия серверной ветки
  origin/feature/cart   ← копия ветки, которой у вас нет локально
```

`origin/main` — это локальное представление того, как выглядела ветка `main` на сервере при последнем `git fetch` или `git pull`.

## Просмотр всех веток (локальных и удалённых)

```bash
git branch -a

# Пример вывода:
# * main                      ← текущая локальная ветка
#   feature/auth              ← другая локальная ветка
#   remotes/origin/main       ← удалённая ветка
#   remotes/origin/feature/auth
#   remotes/origin/feature/cart
```

## Просмотр только удалённых веток

```bash
git branch -r

# Вывод:
#   origin/HEAD -> origin/main
#   origin/main
#   origin/feature/auth
#   origin/feature/cart
```

С информацией о последнем коммите:

```bash
git branch -r -v
#   origin/main       a1b2c3d Last commit message
#   origin/feature/auth b4c5d6e Add authentication
```

## Синхронизация информации с сервером

Git не обновляет информацию об удалённых ветках автоматически. Нужно явно запросить:

```bash
git fetch

# Или для конкретного удалённого репо
git fetch origin
```

После `fetch` вы увидите новые ветки, которые создали коллеги. `git fetch` не изменяет ваши локальные ветки — только обновляет локальные представления удалённых веток (`origin/*`).

```bash
# Обновить информацию и удалить устаревшие ветки
git fetch --prune

# Показать что было обновлено
git fetch -v
```

## Просмотр привязки локальных веток к удалённым

```bash
git branch -vv

# Вывод:
# * main    a1b2c3d [origin/main] Last commit
#   feature b4c5d6e [origin/feature: ahead 2] Add feature
#   local-branch c7d8e9f (нет привязки к удалённой)
```

Значения в квадратных скобках: `[origin/main]` — ветка отслеживает `origin/main`. `ahead 2` — у вас 2 коммита, которых нет на сервере. `behind 1` — на сервере есть 1 коммит, которого у вас нет.

## Просмотр подробной информации об удалённом репо

```bash
git remote show origin

# Вывод:
# * remote origin
#   Fetch URL: git@github.com:user/repo.git
#   Push  URL: git@github.com:user/repo.git
#   HEAD branch: main
#   Remote branches:
#     main    tracked
#     feature/auth  tracked
#     feature/cart  tracked
#   Local branch configured for 'git pull':
#     main merges with remote main
```

## Работа с удалёнными ветками

**Переключиться на удалённую ветку (создать локальную копию):**

```bash
git switch feature/cart        # Git создаст локальную ветку, отслеживающую origin/feature/cart
# или
git checkout -b feature/cart origin/feature/cart  # явный синтаксис
```

**Сравнить локальную ветку с удалённой:**

```bash
git diff main origin/main
```

**Просмотр коммитов в удалённой ветке:**

```bash
git log origin/feature/auth --oneline
```

**Удалить устаревшие локальные представления удалённых веток:**

```bash
# Просмотр что будет удалено
git remote prune origin --dry-run

# Удалить
git remote prune origin

# Или при fetch:
git fetch --prune
```

## Практические примеры

```bash
# Просмотр всех веток (локальные и удалённые)
git branch -a

# Просмотр только удалённых веток
git branch -r

# С информацией о последних коммитах
git branch -r -v

# Просмотр локальных веток с информацией об отслеживании
git branch -vv

# Обновление информации об удалённых ветках
git fetch

# Обновление с удалением устаревших
git fetch --prune

# Все удалённые репозитории
git remote -v

# Коммиты в удалённой ветке
git log origin/main

# Сравнение локальной ветки с удалённой
git diff main origin/main

# Ветки, которые уже удалены на сервере
git remote prune origin --dry-run

# Подробная информация об удалённом репо
git remote show origin
```

## Отслеживание удалённых веток

Локальная ветка может отслеживать удалённую ветку. Это значит, что `git pull` и `git push` будут автоматически использовать эту удалённую ветку:

```bash
# При создании ветки из удалённой она автоматически отслеживает её
git switch feature/auth
# Branch 'feature/auth' set up to track 'origin/feature/auth'

# Проверить какие ветки отслеживают какие
git branch -vv
# * main    a1b2c3d [origin/main: up to date]
#   feature b4c5d6e [origin/feature/auth: ahead 2]
#   local   c7d8e9f (no tracking information)

# Информация в скобках:
# [origin/main] = отслеживает origin/main, синхронизирована
# [origin/feature/auth: ahead 2] = впереди на 2 коммита
# [origin/feature/auth: behind 1] = отстаёт на 1 коммит
# (no tracking information) = не отслеживает никакую удалённую ветку
```

## Установка отслеживания после создания ветки

Если ветка создана без отслеживания:

```bash
# Создать локальную ветку без отслеживания
git switch -c local-branch

# Позже установить отслеживание
git branch --set-upstream-to=origin/main
# Branch 'local-branch' set up to track 'origin/main'

# Или с помощью push -u
git push -u origin local-branch
# Branch 'local-branch' set up to track 'origin/local-branch'

# Теперь git pull и git push знают куда отправлять
git pull     # = git pull origin local-branch
git push     # = git push origin local-branch
```

## Удаление удалённых веток

Часто нужно удалить ветку с сервера:

```bash
# Удалить ветку на сервере
git push origin --delete feature/old-feature
# или
git push origin -d feature/old-feature

# Удалить несколько веток
git push origin -d feature/auth feature/cart feature/checkout

# Удалить все локальные представления удалённых веток которых больше нет
git remote prune origin

# Или при fetch
git fetch --prune
```

## Удаление устаревших локальных представлений удалённых веток

После удаления ветки на сервере, локальное представление может остаться:

```bash
# Посмотреть что будет удалено (сухой запуск)
git remote prune origin --dry-run
# Pruning origin
# * [would prune] origin/old-feature
# * [would prune] origin/deleted-branch

# Реально удалить
git remote prune origin

# Или при fetch автоматически удалить
git fetch --prune
git fetch -p    # сокращённо

# Конфигурировать git чтобы всегда удалять при fetch
git config --global fetch.prune true
# Или только для этого репо
git config fetch.prune true
```

## Проверка отставания/опережения

```bash
# Какие коммиты в удалённой ветке но не в локальной
git log origin/main..main
# Показывает коммиты которые нужно push

# Какие коммиты в локальной но не в удалённой
git log main..origin/main
# Показывает коммиты которые нужно pull

# Более удобный вывод
git log --oneline origin/main..main
git log --oneline main..origin/main

# Количество отличающихся коммитов
git rev-list --count main..origin/main
git rev-list --count origin/main..main

# Информация из git branch -vv проще и понятнее для обзора
git branch -vv
# * main    a1b2c3d [origin/main: behind 5]
```

## Работа с удалёнными веткми при командной разработке

Практический пример работы в команде:

```bash
# Ваш коллега создал новую ветку feature/payment на сервере
# Вы не видите её локально

# Обновить информацию с сервера
git fetch

# Теперь видите новую ветку
git branch -r
# * origin/main
#   origin/feature/auth
#   origin/feature/payment    ← новая!

# Переключиться на её локальную копию
git switch feature/payment
# Branch 'feature/payment' set up to track 'origin/feature/payment'

# Сделать свои изменения
echo "my changes" >> feature.js
git add feature.js
git commit -m "Add payment feature enhancement"

# Отправить обновленную ветку
git push

# Ваш коллега получает уведомление что вы push-нули в feature/payment
# Он делает git fetch и видит ваши изменения
```

## Сравнение веток (локальной и удалённой)

```bash
# Посмотреть разницу между веткой и её удалённой версией
git diff main origin/main

# Более детально
git diff --stat main origin/main

# Показать все коммиты разницы
git log -p main..origin/main

# Список изменённых файлов
git diff --name-only main origin/main

# Просмотр specific file
git diff main origin/main -- src/auth.js
```

## Синхронизация локальных веток с удалёнными

```bash
# Fast-forward слияние (если только добавлены коммиты на сервере)
git merge origin/main
# это безопасно, не создаёт merge коммит если возможен ff

# Или через rebase (переписать историю локальных коммитов)
git rebase origin/main
# Переместит ваши коммиты на top удалённых

# Pull = fetch + merge
git pull

# Pull с rebase вместо merge
git pull --rebase

# Конфигурировать pull всегда делать rebase
git config pull.rebase true
```

## Часто задаваемые вопросы

**Что означает origin в удалённых ветках?** Origin — это стандартное имя основного удалённого репозитория. Устанавливается при `git clone`. Можно иметь несколько удалённых репо с разными именами (origin, upstream и т.д.).

**Почему я не вижу ветку коллеги в `git branch -r`?** Потому что информация устарела. Запустите `git fetch` для синхронизации с сервером.

**Как скачать удалённую ветку?** `git switch имя-ветки` — Git автоматически создаст локальную ветку, отслеживающую одноимённую удалённую.

**Какая разница между git fetch и git pull?** `git fetch` только обновляет информацию об удалённых ветках, не меняя ваши локальные ветки. `git pull` = `git fetch` + `git merge`.

**Как удалить локальное представление удалённой ветки?** `git remote prune origin` удалит устаревшие `origin/*` ветки, которых больше нет на сервере.

**Что означает отслеживание ветки?** Отслеживание означает что локальная ветка привязана к удалённой ветке. Тогда `git pull` и `git push` автоматически знают с какой удалённой веткой работать.

## Заключение

Для просмотра удалённых веток: `git branch -a` (все ветки) или `git branch -r` (только удалённые). Не забывайте делать `git fetch` перед просмотром — иначе информация может быть устаревшей.

Следующие шаги: [переключение на удалённую ветку]({{< relref "pereklyuchitsya-mezhdu-vetkami" >}}), [git fetch]({{< relref "git-fetch" >}}), [git pull]({{< relref "git-pull" >}}).
