---
title: "Git субмодули: репозиторий внутри репозитория"
description: "Полное руководство по git submodules. Добавление, клонирование, обновление субмодулей. Команды git submodule add, init, update. Альтернатива git subtree."
date: 2026-01-31
lastmod: 2026-03-14
draft: false
slug: "git-submodules"
keywords: ["git submodules", "субмодули git", "git submodule add", "git submodule change branch", "git submodule change url", "git submodule set-branch", "git submodule vs subtree", "does not have a commit checked out"]
tags: ["git", "advanced"]
categories: ["git"]
---

Git субмодуль — это способ включить один Git репозиторий в другой. Подпроект остаётся отдельным репозиторием с собственной историей, но ссылка на конкретный коммит хранится в родительском репозитории.

## Когда использовать субмодули

```
Хорошие случаи:
- Общая библиотека, используемая в нескольких проектах
- Сторонняя зависимость, которую нужно кастомизировать
- Документация или ресурсы в отдельном репозитории
- Монорепозиторий с независимыми компонентами

Плохие случаи:
- Зависимости, которые легко управлять через npm/pip/cargo
- Проекты с частыми обновлениями зависимостей
- Команды, незнакомые с субмодулями (добавляет сложность)
```

## Добавление субмодуля

```bash
# Добавить субмодуль в папку vendor/my-lib
git submodule add git@github.com:user/my-lib.git vendor/my-lib

# Добавить с конкретной веткой
git submodule add -b main git@github.com:user/my-lib.git vendor/my-lib

# Git создаёт:
# - vendor/my-lib/ (содержимое субмодуля)
# - .gitmodules (файл конфигурации субмодулей)

# Просмотреть .gitmodules
cat .gitmodules
# [submodule "vendor/my-lib"]
#     path = vendor/my-lib
#     url = git@github.com:user/my-lib.git

# Зафиксировать добавление субмодуля
git commit -m "Add my-lib as submodule"
```

## Клонирование репозитория с субмодулями

```bash
# Способ 1: клонировать и инициализировать субмодули отдельно
git clone git@github.com:user/parent-project.git
cd parent-project
git submodule init      # зарегистрировать субмодули
git submodule update    # клонировать субмодули

# Способ 2: клонировать с субмодулями сразу
git clone --recurse-submodules git@github.com:user/parent-project.git

# Если уже клонировали без субмодулей
git submodule update --init --recursive
# --init: регистрирует субмодули
# --recursive: рекурсивно для вложенных субмодулей
```

## Обновление субмодулей

```bash
# Обновить все субмодули до последнего коммита в tracked ветке
git submodule update --remote

# Обновить конкретный субмодуль
git submodule update --remote vendor/my-lib

# Обновить и рекурсивно (если субмодули содержат субмодули)
git submodule update --remote --recursive

# После обновления — зафиксировать изменённые ссылки
git add vendor/my-lib
git commit -m "Update my-lib to latest"
```

## Работа внутри субмодуля

```bash
# Перейти в субмодуль
cd vendor/my-lib

# Субмодуль изначально в detached HEAD
git status
# HEAD detached at a1b2c3d

# Переключиться на ветку
git switch main

# Внести изменения и закоммитить
git commit -am "Fix bug in library"
git push origin main

# Вернуться в родительский проект
cd ../..

# Зафиксировать новую ссылку
git add vendor/my-lib
git commit -m "Update my-lib: fix bug"
```

## Статус субмодулей

```bash
# Посмотреть статус субмодулей
git submodule status
# a1b2c3d vendor/my-lib (v1.0-2-ga1b2c3d)
# ^ = указывает на коммит
# + = есть незафиксированные изменения

# Краткий статус
git submodule summary

# Полная информация
git submodule foreach 'git log --oneline -1'
```

## Изменение ветки субмодуля

По умолчанию субмодуль не привязан к конкретной ветке — он фиксируется на конкретный коммит. Чтобы субмодуль отслеживал определённую ветку и автоматически обновлялся до её последнего коммита:

```bash
# Привязать субмодуль к ветке (git submodule set-branch)
git submodule set-branch --branch develop vendor/my-lib
# Теперь .gitmodules содержит: branch = develop

# Проверить результат
cat .gitmodules
# [submodule "vendor/my-lib"]
#     path = vendor/my-lib
#     url = git@github.com:user/my-lib.git
#     branch = develop

# Обновить субмодуль до HEAD выбранной ветки
git submodule update --remote vendor/my-lib

# Зафиксировать изменение ветки
git add .gitmodules vendor/my-lib
git commit -m "Switch my-lib submodule to develop branch"
```

Альтернативный способ — отредактировать `.gitmodules` вручную:

```ini
[submodule "vendor/my-lib"]
    path = vendor/my-lib
    url = git@github.com:user/my-lib.git
    branch = develop
```

Затем синхронизировать и обновить:

```bash
git submodule sync vendor/my-lib
git submodule update --remote vendor/my-lib
```

## Изменение URL субмодуля

Если репозиторий субмодуля переехал на другой адрес (переименован, сменился хостинг), нужно обновить URL:

```bash
# Изменить URL в конфигурации
git submodule set-url vendor/my-lib git@github.com:new-org/my-lib.git

# Проверить .gitmodules
cat .gitmodules

# Синхронизировать локальную конфигурацию с .gitmodules
git submodule sync vendor/my-lib
# или для всех субмодулей:
git submodule sync

# Зафиксировать изменение URL
git add .gitmodules
git commit -m "Update my-lib submodule URL"
```

Если нужно изменить только локально (без коммита) — отредактируйте `.git/config`:

```bash
git config submodule.vendor/my-lib.url git@github.com:new-org/my-lib.git
```

## Типичные ошибки субмодулей

### "does not have a commit checked out"

```
fatal: 'vendor/my-lib' does not have a commit checked out
```

Причина: субмодуль добавлен в репозиторий, но не инициализирован/не клонирован.

```bash
# Решение: инициализировать и клонировать
git submodule update --init vendor/my-lib
# или все сразу:
git submodule update --init --recursive
```

### "fatal: adding files failed" / "please stage your changes to .gitmodules"

```
fatal: please stage your changes to .gitmodules or stash them to proceed
```

Причина: незакоммиченные изменения в `.gitmodules`.

```bash
# Добавить .gitmodules в staging
git add .gitmodules
git commit -m "Update submodule config"
# или если хотите отменить:
git restore .gitmodules
```

### "warning: adding embedded git repository"

```
warning: adding embedded git repository: vendor/my-lib
hint: If you meant to add a submodule, use:
hint:   git submodule add <url> vendor/my-lib
```

Причина: попытка сделать `git add` на папку с `.git` внутри. Git обнаружил вложенный репозиторий.

```bash
# Правильный способ добавить субмодуль:
git submodule add git@github.com:user/my-lib.git vendor/my-lib
# Не через git add vendor/my-lib
```

### Субмодуль показывает старый коммит после git pull

```bash
# git pull обновляет ссылку в родительском репозитории,
# но не скачивает содержимое субмодуля автоматически

# Обновить субмодули после pull
git pull
git submodule update --init --recursive

# Или настроить автоматическое обновление:
git config --global submodule.recurse true
```

### commit or discard the untracked or modified content in submodules

```
error: The following untracked working tree files would be overwritten by checkout:
  vendor/my-lib/generated.js
```

Внутри субмодуля есть незакоммиченные изменения. Перейдите в субмодуль и либо закоммитьте, либо отмените изменения:

```bash
cd vendor/my-lib
git status
git restore .   # отменить изменения
# или
git stash       # спрятать изменения
cd ../..
git submodule update
```

## Полезные команды для работы с субмодулями

```bash
# Выполнить команду в каждом субмодуле
git submodule foreach 'git status'
git submodule foreach 'git pull origin main'
git submodule foreach --recursive 'git clean -fd'

# Список всех субмодулей
git submodule
cat .gitmodules

# Информация о конкретном субмодуле
git submodule status vendor/my-lib

# Клонировать рекурсивно (включая субмодули субмодулей)
git clone --recurse-submodules --shallow-submodules <url>
```

## Удаление субмодуля

Удаление субмодуля — многошаговый процесс:

```bash
# Шаг 1: Зарегистрировать удаление в .gitmodules
git submodule deinit vendor/my-lib

# Шаг 2: Удалить запись из .git/config
git rm vendor/my-lib

# Шаг 3: Удалить кешированные данные
rm -rf .git/modules/vendor/my-lib

# Шаг 4: Закоммитить
git commit -m "Remove my-lib submodule"
```

## Альтернатива: git subtree

`git subtree` — более простой способ включить один репозиторий в другой:

```bash
# Добавить subtree
git subtree add --prefix vendor/my-lib \
  git@github.com:user/my-lib.git main --squash

# Обновить subtree
git subtree pull --prefix vendor/my-lib \
  git@github.com:user/my-lib.git main --squash

# Отправить изменения обратно
git subtree push --prefix vendor/my-lib \
  git@github.com:user/my-lib.git main
```

**Submodule vs Subtree:**

```
Submodule:
+ Чёткое разделение — субмодуль это отдельный репозиторий
+ Не дублирует историю
- Сложнее для разработчиков (нужно знать команды субмодулей)
- Клонирование требует --recurse-submodules
- Проблемы при pull если не обновить субмодули

Subtree:
+ Проще в использовании (обычный git clone/pull)
+ Код хранится прямо в родительском репозитории
- Дублирует историю (смешанная история)
- Сложнее отправлять изменения обратно
```

## Часто задаваемые вопросы

**Почему папка субмодуля пустая после клонирования?** Субмодули не клонируются автоматически. Выполните `git submodule update --init --recursive` или клонируйте с флагом `--recurse-submodules`.

**Как обновить все субмодули сразу?** `git submodule update --remote --recursive` обновит все до последнего коммита в их tracking ветках.

**Почему субмодуль показывает detached HEAD?** Субмодуль фиксируется на конкретный коммит, не на ветку. Это нормально. Для разработки — переключитесь на ветку внутри субмодуля: `cd vendor/my-lib && git switch main`.

**Как передать --recurse-submodules по умолчанию?** `git config --global submodule.recurse true` — автоматически обновляет субмодули при `git pull`.

**Когда лучше subtree, а когда submodule?** Submodule — когда подпроект активно разрабатывается отдельно. Subtree — когда нужна простота и команда не хочет изучать субмодули. Для простых зависимостей — лучше менеджер пакетов (npm, pip).

## Заключение

Git субмодули позволяют включить один репозиторий в другой с чётким разделением истории. Добавление: `git submodule add <url> <path>`. Клонирование с субмодулями: `git clone --recurse-submodules`. Обновление: `git submodule update --remote`. Субмодули добавляют сложность — рассмотрите git subtree или менеджер пакетов как альтернативу.

## По теме

- [Удалить субмодуль]({{< relref "git-udalit-submodul" >}})
- [git clone]({{< relref "git-clone" >}})
- [Bare-репозиторий]({{< relref "git-init-bare" >}})
- [Собственный git-сервер]({{< relref "sobstvennyj-git-server" >}})
