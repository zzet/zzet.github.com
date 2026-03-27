---
title: "git prune: удаление недостижимых объектов и устаревших ссылок"
description: "Команда git prune для очистки репозитория. Удаление недостижимых объектов, git fetch --prune для синхронизации удалённых веток, git remote prune."
date: 2026-01-21
lastmod: 2026-01-21
draft: false
slug: "git-prune"
keywords: ["git prune", "git fetch prune", "удаление устаревших веток git", "git remote prune origin", "очистить ссылки git", "git gc prune"]
tags: ["git", "intermediate"]
categories: ["git"]
---

`git prune` удаляет из локального хранилища объекты, которые недостижимы ни из одной ссылки (ветки, теги, HEAD). В повседневной работе чаще используется `git fetch --prune` — для синхронизации локальных отслеживающих веток с удалённым репозиторием.

## Что такое недостижимые объекты

При работе с Git появляются «осиротевшие» объекты — коммиты, деревья и файлы, которые больше не доступны ни из одной активной ветки:

```bash
# Ситуации, создающие недостижимые объекты:
git commit --amend          # старый коммит остаётся в объектах
git reset HEAD~1            # «удалённый» коммит остаётся в объектах
git rebase main             # старые коммиты ветки остаются
git branch -D feature       # коммиты ветки остаются (если не слиты)

# Проверить недостижимые объекты
git fsck --unreachable
# unreachable commit a1b2c3d4...
# unreachable tree   e5f6a7b8...
# unreachable blob   c9d0e1f2...
```

## git prune: очистка объектов

`git prune` удаляет объекты, недоступные ни из одной ссылки и старше grace period:

```bash
# Показать что будет удалено (без реального удаления)
git prune --dry-run
# Would prune a1b2c3d4...
# Would prune e5f6a7b8...

# Удалить недостижимые объекты
git prune

# Удалить объекты старше 1 часа (обход grace period)
git prune --expire=1.hour.ago

# Удалить объекты старше конкретной даты
git prune --expire="2026-01-01"

# С подробным выводом
git prune -v
# Removing a1b2c3d4...
```

Git по умолчанию защищает недостижимые объекты в течение grace period (14 дней). Это позволяет восстановить случайно сброшенные коммиты через `git reflog`.

## git gc: предпочтительный способ очистки

В большинстве случаев лучше использовать `git gc`, а не `git prune` напрямую:

```bash
# git gc включает git prune и дополнительную оптимизацию
git gc

# Что делает git gc:
# 1. git pack-refs — упаковывает ссылки
# 2. git reflog expire — удаляет старые записи reflog
# 3. git pack-objects — создаёт pack-файлы
# 4. git prune-packed — удаляет свободные объекты уже в pack-файлах
# 5. git prune — удаляет недостижимые объекты

# Агрессивная оптимизация (медленнее, лучше сжатие)
git gc --aggressive

# Только прune без упаковки
git gc --prune=now  # удалить все недостижимые объекты немедленно
git gc --prune=all  # то же самое
```

## git fetch --prune: синхронизация удалённых веток

Самый частый случай использования прунинга — удаление локальных копий удалённых веток, которые были удалены на сервере:

```bash
# Ситуация: ветка feature/old удалена на GitHub
# Локально она всё ещё есть как origin/feature/old

# Проверить устаревшие ссылки
git remote show origin
# Remote branches:
#   main             tracked
#   feature/old      stale (use 'git remote prune' to remove)

# Удалить конкретный remote
git remote prune origin
# Pruning origin
# URL: git@github.com:user/project.git
#  * [pruned] origin/feature/old

# Или при fetch автоматически удалять устаревшие
git fetch --prune
git fetch -p       # короткий вариант

# Настроить автоматический прунинг при каждом fetch
git config --global fetch.prune true
# Теперь git fetch всегда будет удалять устаревшие ссылки
```

## Сравнение команд

```bash
# git prune
# Удаляет ОБЪЕКТЫ (blob, tree, commit) недостижимые из refs
# Работает с .git/objects/
# Обычно вызывается через git gc

# git remote prune <remote>
# Удаляет ЛОКАЛЬНЫЕ ССЫЛКИ на удалённые ветки (refs/remotes/)
# Которые больше не существуют на удалённом сервере

# git fetch --prune
# То же что git remote prune, но во время fetch
# Автоматически синхронизирует refs/remotes/

# git prune-packed
# Удаляет свободные объекты, которые уже есть в pack-файлах
# Вызывается автоматически через git gc
```

## Практические сценарии

```bash
# Сценарий 1: в команде удаляют ветки после merge PR
# После git fetch появляются устаревшие origin/* ветки

git branch -r
# origin/main
# origin/feature/auth      ← уже удалена на GitHub
# origin/feature/payments  ← уже удалена на GitHub
# origin/hotfix/login      ← уже удалена на GitHub

git fetch --prune
# - [deleted]         (none)     -> origin/feature/auth
# - [deleted]         (none)     -> origin/feature/payments
# - [deleted]         (none)     -> origin/hotfix/login

# Сценарий 2: после серии rebase репозиторий разрастается
du -sh .git/objects/
# 45M .git/objects/

git gc --prune=now
# Counting objects: 234, done.
# Delta compression...
# Removing unreachable objects...

du -sh .git/objects/
# 12M .git/objects/

# Сценарий 3: найти что занимает место
git count-objects -v
# count: 1234        ← свободные объекты
# size: 45678        ← в килобайтах
# in-pack: 5678
# packs: 3
# size-pack: 12345
# prune-packable: 234  ← объекты в pack, но ещё есть свободные копии

git gc  # упакует и очистит
```

## Настройка автоматической очистки

```bash
# Автоматический gc при накоплении объектов
git config --global gc.auto 6700
# При > 6700 свободных объектов запускается git gc --auto

# Отключить автоматический gc
git config --global gc.auto 0

# Настроить срок жизни недостижимых объектов
git config --global gc.pruneExpire "14.days.ago"

# Настроить fetch.prune глобально
git config --global fetch.prune true
git config --global remote.origin.prune true  # для конкретного remote
```

## git prune-tags

Удаление локальных тегов, не существующих на удалённом:

```bash
# Удалить устаревшие локальные теги (аналог --prune для тегов)
git fetch --prune --prune-tags

# Или только теги
git fetch origin --prune-tags
```

## Часто задаваемые вопросы

**Можно ли восстановить объекты после git prune?** Только если они ещё в reflog. После `git prune` объекты удалены физически. Вот почему grace period по умолчанию 14 дней.

**git prune удалит мои коммиты из веток?** Нет. `git prune` удаляет только объекты, недостижимые из ЛЮБОЙ ссылки (ветки, теги, HEAD, stash, reflog). Коммиты в ветках всегда достижимы.

**Почему git remote prune не удаляет локальные ветки?** Потому что локальные ветки (refs/heads/) независимы от удалённых (refs/remotes/). `git remote prune` удаляет только tracking references. Локальные ветки нужно удалять вручную: `git branch -d old-feature`.

**Как часто нужно делать git gc?** Git запускает gc автоматически при накоплении достаточного количества объектов. Явный `git gc` нужен только если репозиторий вырос или нужно срочно освободить место.

**Что такое grace period и зачем он нужен?** 14 дней по умолчанию. Если вы случайно сбросили коммит через `git reset`, вы можете найти его в `git reflog` и восстановить. После grace period объект удаляется и восстановление невозможно.

## Заключение

`git prune` очищает недостижимые объекты из локального хранилища Git. В повседневной практике используйте `git gc` (включает prune) и `git fetch --prune` (синхронизирует удалённые ветки). Настройте `fetch.prune true` глобально, чтобы автоматически удалять устаревшие удалённые ветки при каждом `git fetch`.

## По теме

- [git reflog]({{< relref "git-reflog" >}})
- [Объекты Git изнутри]({{< relref "git-internals-objects" >}})
- [Удалённые ветки]({{< relref "udalennye-vetki-git" >}})
- [git remote -v]({{< relref "git-remote-v" >}})
- [git fetch]({{< relref "git-fetch" >}})
