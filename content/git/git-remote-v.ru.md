---
title: "git remote: управление удалёнными репозиториями — полное руководство"
description: "git remote -v, set-url, add, remove, rename, show origin. Переключение HTTPS↔SSH. Ошибка remote already exists."
date: 2026-01-26
lastmod: 2026-03-14
draft: false
slug: "git-remote-v"
keywords: ["git remote v что делает", "git remote -v", "git remote show origin", "git remote set-url", "git remote change origin", "remote origin already exists", "git remote remove", "удалить remote git", "изменить url remote git"]
tags: ["git", "remote", "beginner"]
categories: ["git"]
---

`git remote -v` — одна из первых команд, которую нужно выполнить при работе с чужим репозиторием или при отладке проблем с push/pull. Она показывает все удалённые репозитории и их URL.

## Что такое удалённые репозитории

Удалённый репозиторий (remote) — это копия вашего проекта на сервере. При клонировании через `git clone` автоматически создаётся одна запись с именем `origin`, указывающая на клонированный адрес.

При форке проекта обычно добавляют второй remote — `upstream` — ссылку на оригинальный репозиторий. Стандартные имена: `origin` (ваш форк или ваш репо), `upstream` (оригинальный проект).

## Синтаксис и вывод команды

```bash
# Просмотр всех удалённых репо с URL
git remote -v

# Пример вывода для обычного репо:
# origin  https://github.com/username/repo.git (fetch)
# origin  https://github.com/username/repo.git (push)

# Пример вывода для fork:
# origin    https://github.com/myusername/react.git (fetch)
# origin    https://github.com/myusername/react.git (push)
# upstream  https://github.com/facebook/react.git (fetch)
# upstream  https://github.com/facebook/react.git (push)
```

Каждый remote показывается дважды: одна строка для `fetch` (получение обновлений), другая для `push` (отправка изменений). Обычно они совпадают, но можно настроить разные URL для получения и отправки.

## Разница между git remote и git remote -v

```bash
# Только имена удалённых репо (без URL)
git remote
# Вывод:
# origin
# upstream

# Имена + URL (verbose/подробный режим)
git remote -v
# Вывод:
# origin    https://github.com/user/repo.git (fetch)
# origin    https://github.com/user/repo.git (push)
# upstream  https://github.com/original/repo.git (fetch)
# upstream  https://github.com/original/repo.git (push)
```

Флаг `-v` означает `--verbose` (подробный вывод).

## Интерпретация результатов

**HTTPS URL:** `https://github.com/username/repo.git`
- Требует ввода пароля (или токена) при push
- Работает без дополнительной настройки

**SSH URL:** `git@github.com:username/repo.git`
- Требует настроенного SSH-ключа
- Удобнее для частого использования — не требует ввода пароля

Если видите SSH URL в `git remote -v`, но push не работает — возможно, SSH-ключ не настроен. Если видите HTTPS — нужен Personal Access Token.

## Практические сценарии

**Проверка перед push:**
```bash
# Убедиться, что отправляем в нужный репо
git remote -v
git push origin main
```

**Диагностика проблем:**
```bash
# Неправильный URL? Исправить
git remote -v
git remote set-url origin https://github.com/correct-user/repo.git
git remote -v  # Проверить изменение
```

**Переключение с HTTPS на SSH:**
```bash
# Посмотреть текущий URL
git remote -v
# origin  https://github.com/user/repo.git (fetch)

# Изменить на SSH
git remote set-url origin git@github.com:user/repo.git

# Проверить
git remote -v
# origin  git@github.com:user/repo.git (fetch)
```

## git remote show origin — подробная информация

`git remote show` выдаёт полную информацию о remote: URL, текущая ветка, список веток с tracking статусом:

```bash
git remote show origin
# * remote origin
#   Fetch URL: git@github.com:user/project.git
#   Push  URL: git@github.com:user/project.git
#   HEAD branch: main
#   Remote branches:
#     main           tracked
#     develop        tracked
#     feature/auth   tracked
#   Local branch configured for 'git pull':
#     main merges with remote main
#   Local ref configured for 'git push':
#     main pushes to main (up to date)
```

Отличие от `git remote -v`: show даёт статус веток и настройки pull/push, а не просто URL.

## Изменение URL remote: git remote set-url

Используется когда репозиторий переехал, или нужно переключиться с HTTPS на SSH:

```bash
# Изменить URL на HTTPS
git remote set-url origin https://github.com/newuser/repo.git

# Изменить URL на SSH
git remote set-url origin git@github.com:newuser/repo.git

# Проверить изменение
git remote -v
```

### Переключение с HTTPS на SSH (популярный случай)

```bash
# До: HTTPS (требует токен при каждом push)
git remote -v
# origin  https://github.com/user/repo.git (fetch)
# origin  https://github.com/user/repo.git (push)

# Переключить на SSH (не требует пароля если настроен SSH ключ)
git remote set-url origin git@github.com:user/repo.git

# После
git remote -v
# origin  git@github.com:user/repo.git (fetch)
# origin  git@github.com:user/repo.git (push)
```

### Задать разные URL для fetch и push

```bash
# Установить отдельный URL только для push
git remote set-url --push origin git@github.com:user/repo.git

# Теперь fetch и push используют разные URL
git remote -v
# origin  https://github.com/user/repo.git (fetch)
# origin  git@github.com:user/repo.git (push)
```

## Удаление remote: git remote remove

```bash
# Удалить remote из конфигурации
git remote remove upstream
git remote rm upstream    # сокращённая форма

# Проверить что удалился
git remote -v
```

Удаление remote не удаляет локальные ветки и не трогает историю — только убирает запись из `.git/config`. Все скачанные ветки типа `upstream/main` также очищаются.

## Переименование remote: git remote rename

```bash
# Переименовать origin в old-origin (при добавлении нового origin)
git remote rename origin old-origin

# Переименовать upstream в source
git remote rename upstream source

# Проверить
git remote -v
```

После переименования все tracking-ветки обновляются автоматически: `origin/main` → `old-origin/main`.

## Ошибка: remote origin already exists

```
error: remote origin already exists.
```

Возникает когда пытаетесь добавить remote с именем, которое уже существует:

```bash
git remote add origin https://github.com/user/repo.git
# error: remote origin already exists.
```

Решения:

```bash
# Вариант 1: изменить URL существующего remote (не добавлять новый)
git remote set-url origin https://github.com/user/repo.git

# Вариант 2: удалить и добавить заново
git remote remove origin
git remote add origin https://github.com/user/repo.git

# Вариант 3: добавить под другим именем
git remote add origin2 https://github.com/user/repo.git
```

Проверить что сейчас настроено:

```bash
git remote -v
```

## Добавление нескольких remote

Стандартная конфигурация для работы с форком:

```bash
# origin — ваш форк
git remote add origin git@github.com:myusername/react.git

# upstream — оригинальный проект
git remote add upstream git@github.com:facebook/react.git

# Получить ветки оригинала
git fetch upstream

# Обновить свой main из оригинала
git switch main
git merge upstream/main
git push origin main
```

Пуш в несколько remote:

```bash
# Добавить второй push URL к существующему remote
git remote set-url --add --push origin git@github.com:mirror/repo.git
# Теперь git push отправляет в оба адреса
```

## Быстрая шпаргалка по git remote

```bash
git remote -v                          # список всех remote с URL
git remote show origin                 # полная информация о remote
git remote add <имя> <url>             # добавить новый remote
git remote set-url origin <новый-url>  # изменить URL
git remote rename <старое> <новое>     # переименовать
git remote remove <имя>                # удалить remote
git remote prune origin                # очистить устаревшие ветки
```

## Часто задаваемые вопросы

**Какая разница между `git remote` и `git remote -v`?** `git remote` показывает только имена удалённых репозиториев. `git remote -v` (verbose) добавляет URL для fetch и push операций.

**Почему для одного remote отображаются две строки (fetch и push)?** URL для получения и отправки могут быть разными. Обычно они одинаковы, но настраиваются отдельно через `git remote set-url --push`.

**Что означает "origin" в выводе?** Стандартное имя для основного удалённого репозитория. Назначается автоматически при `git clone`. Можно переименовать: `git remote rename origin <новое-имя>`.

**Как исправить "remote origin already exists"?** Используйте `git remote set-url origin <url>` вместо `git remote add origin <url>` — это изменяет URL существующего remote.

**git remote remove удалит мои локальные ветки?** Нет. Удаляется только запись о remote в конфигурации. Локальные ветки (`main`, `develop`) остаются. Remote-tracking ветки (`origin/main`) удаляются.

## Заключение

`git remote -v` — первый шаг диагностики при проблемах с push/pull. Для изменения URL: `git remote set-url origin <url>`. Для удаления: `git remote remove <имя>`. Ошибка "remote origin already exists" решается через `set-url`, а не повторный `add`.

Для добавления нового remote используйте {{< relref "git-remote-add" >}}.
Для подключения по SSH — {{< relref "podklyuchenie-k-git-po-ssh" >}}.
При проблемах с устаревшими ветками — {{< relref "git-unable-to-update-local-ref" >}}.
