---
title: "Как посмотреть список репозиториев Git"
description: "Как посмотреть список репозиториев git: git remote -v, git ls-remote, поиск локальных репозиториев на диске."
date: 2025-12-06
lastmod: 2025-12-06
draft: false
slug: spisok-repozitoriev-git
keywords:
  - список репозиториев git
  - git список репозиториев
  - git ls-remote
  - git remote список
  - git list repositories
  - посмотреть репозитории git
  - найти git репозитории на компьютере
tags:
  - git
categories:
  - git
---

При работе с Git часто возникает необходимость посмотреть список репозиториев — но что именно вы хотите узнать? Список удалённых репозиториев текущего проекта? Ветки на сервере? Все локальные репозитории на вашем компьютере? В этом руководстве разберём все основные способы получить нужную информацию.

## Что означает "список репозиториев git"

Прежде чем искать репозитории, нужно понять, что именно вам нужно. В Git есть несколько разных концепций, которые люди часто путают:

### Три разных вопроса

**1. Удалённые репозитории текущего проекта** — это ссылки на внешние серверы (GitHub, GitLab и т.д.), которые используются вашим текущим проектом. Обычно их 1-2: `origin` (основной репозиторий) и `upstream` (если вы работаете с форком).

**2. Ветки и объекты удалённого репозитория** — это информация о том, какие ветки существуют на сервере, без необходимости клонировать весь репозиторий. Полезно перед клонированием или при работе с чужим проектом.

**3. Локальные репозитории на вашем диске** — это все папки с `.git` на вашем компьютере. Вы можете иметь десятки проектов, разбросанные по разным местам.

Разберём каждый случай отдельно.

## Список удалённых репозиториев текущего проекта

Если вы уже находитесь внутри git репозитория, получить список удалённых репозиториев очень просто.

### Основные команды

```bash
# Только имена удалённых репозиториев
git remote

# Имена и URL (для fetch и push)
git remote -v

# Детальная информация о конкретном remote
git remote show origin
```

### Пример вывода

```bash
$ git remote
origin
upstream

$ git remote -v
origin          https://github.com/myuser/myproject.git (fetch)
origin          https://github.com/myuser/myproject.git (push)
upstream        https://github.com/original/myproject.git (fetch)
upstream        https://github.com/original/myproject.git (push)
```

Команда `git remote -v` показывает все удалённые репозитории с указанием URL для операций fetch (загрузка) и push (выгрузка).

### Что означает origin и upstream

- **origin** — стандартное имя для основного репозитория, обычно вашего форка на GitHub или GitLab
- **upstream** — репозиторий оригинального проекта, из которого вы сделали форк. Используется для синхронизации с основной веткой разработки

### Несколько remote в одном проекте

Многие разработчики работают по workflow форка:
1. Делают форк оригинального проекта на своём аккаунте
2. Клонируют свой форк (это становится `origin`)
3. Добавляют оригинальный репозиторий как `upstream`

Это позволяет получать обновления из оригинального проекта и одновременно вести свою работу в форке.

```bash
# Добавить upstream после того, как вы склонировали форк
git remote add upstream https://github.com/original/repo.git

# Обновиться из upstream
git fetch upstream
git merge upstream/main
```

## Список веток удалённого репозитория без клонирования

Иногда нужно посмотреть, какие ветки существуют на сервере, не клонируя весь репозиторий. Для этого используется команда `git ls-remote`.

### Базовый синтаксис

```bash
# Посмотреть все ветки и теги удалённого репозитория
git ls-remote https://github.com/user/repo.git

# Только ветки
git ls-remote --heads https://github.com/user/repo.git

# Только теги
git ls-remote --tags https://github.com/user/repo.git

# Если remote уже добавлен, можно использовать его имя
git ls-remote --heads origin
```

### Пример вывода

```bash
$ git ls-remote --heads https://github.com/torvalds/linux.git
27eeba8c91ecfe1b4fb0a7e9a2f9c0f6e2a3b4c5  refs/heads/master
8d4e9f1c3b5a7d2e9f1c3b5a7d2e9f1c3b5a7d2e  refs/heads/next
5f2e1d3c9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d  refs/heads/fixes

$ git ls-remote --tags https://github.com/torvalds/linux.git | head -5
0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b  refs/tags/v6.1
1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d  refs/tags/v6.1-rc1
2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e  refs/tags/v6.2
```

### Что означает HEAD

В выводе вы можете увидеть строку вроде `abcd1234... HEAD -> refs/heads/main`. Это указывает, какая ветка является веткой по умолчанию на сервере.

### Практическое применение

Эта команда полезна для:
- Проверки, какие ветки доступны перед клонированием
- Проверки версий перед загрузкой
- Скриптов автоматизации, которым нужна информация о сервере без полного клонирования

## Найти все локальные git репозитории на диске (Linux/macOS)

Если вы не помните, где находятся ваши проекты, можно найти все репозитории на компьютере.

### Простой способ

```bash
# Найти все папки .git в домашней директории (не глубже 5 уровней)
find ~ -name ".git" -type d -maxdepth 5 2>/dev/null

# Показать только пути к самим репозиториям (без .git)
find ~ -name ".git" -type d -maxdepth 5 2>/dev/null | sed 's/\/.git//'
```

### Почему maxdepth важен

Параметр `-maxdepth 5` ограничивает глубину поиска 5 уровнями вложенности. Это важно, потому что:
- Без этого поиск может занять очень долгое время
- В каждом репозитории может быть огромное количество файлов и папок
- Вы вряд ли разместите проекты глубже чем на 5 уровней от домашней директории

### Красивый вывод со скриптом

```bash
#!/bin/bash
# Скрипт для красивого отображения всех git репозиториев

echo "Git репозитории на вашем компьютере:"
echo "====================================="

find ~ -name ".git" -type d -maxdepth 5 2>/dev/null | while read gitdir; do
    repo_path="${gitdir%/.git}"
    echo ""
    echo "📁 $repo_path"

    # Показать текущую ветку
    if [ -f "$gitdir/HEAD" ]; then
        branch=$(cat "$gitdir/HEAD" | grep -oP 'ref: refs/heads/\K.*')
        echo "   Ветка: $branch"
    fi

    # Показать URL origin если он есть
    if [ -f "$gitdir/config" ]; then
        origin=$(grep -A1 '\[remote "origin"\]' "$gitdir/config" | grep url | awk '{print $3}')
        [ -n "$origin" ] && echo "   Origin: $origin"
    fi
done
```

## Поиск локальных репозиториев на Windows (PowerShell)

На Windows можно использовать PowerShell для аналогичного результата:

```powershell
# Найти все репозитории в профиле пользователя
Get-ChildItem -Path $env:USERPROFILE -Recurse -Filter ".git" -Directory -ErrorAction SilentlyContinue |
  ForEach-Object {
    $repo = $_.Parent.FullName
    Write-Host "📁 $repo"
  }

# Вариант с подробной информацией
Get-ChildItem -Path $env:USERPROFILE -Recurse -Filter ".git" -Directory -ErrorAction SilentlyContinue |
  ForEach-Object {
    $gitDir = $_.FullName
    $repoPath = $_.Parent.FullName
    Write-Host "`n📁 $repoPath"

    # Показать текущую ветку
    $headFile = Join-Path $gitDir "HEAD"
    if (Test-Path $headFile) {
      $branch = (Get-Content $headFile) -replace 'ref: refs/heads/', ''
      Write-Host "   Ветка: $branch"
    }
  }
```

## Список своих репозиториев на GitHub

Если вы хотите увидеть все репозитории своего аккаунта на GitHub, есть несколько способов.

### Способ 1: GitHub CLI (рекомендуется)

GitHub CLI (`gh`) — это официальный инструмент GitHub и самый удобный способ:

```bash
# Список всех репозиториев
gh repo list

# С ограничением на 100 репозиториев
gh repo list --limit 100

# Только собственные репозитории (без форков)
gh repo list --source

# Только форки
gh repo list --fork

# Репозитории конкретной организации
gh repo list myorganization

# Только приватные репозитории
gh repo list --visibility private
```

### Способ 2: GitHub API без CLI

Если GitHub CLI не установлен, можно использовать API напрямую:

```bash
# Требуется личный access token (создаётся в settings/tokens)
curl -s -H "Authorization: token YOUR_GITHUB_TOKEN" \
  "https://api.github.com/user/repos?per_page=100&page=1" | \
  python3 -c "import sys, json; repos = json.load(sys.stdin);
              [print(f\"{r['full_name']} - {r['description'] or 'без описания'}\") for r in repos]"
```

## Список репозиториев на GitLab

Для GitLab процесс похож, но с использованием других инструментов.

### GitLab CLI

```bash
# Список репозиториев текущего пользователя
glab repo list

# С опциями
glab repo list --limit 100
glab repo list --owned  # только собственные
```

### GitLab API

```bash
# Все проекты пользователя (требуется personal access token)
curl "https://gitlab.com/api/v4/projects?owned=true&per_page=100" \
     -H "PRIVATE-TOKEN: YOUR_GITLAB_TOKEN" | \
     python3 -c "import sys, json; repos = json.load(sys.stdin);
                 [print(r['path_with_namespace']) for r in repos]"

# Проекты конкретной группы/организации
curl "https://gitlab.com/api/v4/groups/GROUP_ID/projects?per_page=100" \
     -H "PRIVATE-TOKEN: YOUR_GITLAB_TOKEN"
```

## Управление удалёнными репозиториями

Основные операции для управления remote:

```bash
# Добавить новый remote
git remote add upstream https://github.com/original/repo.git

# Переименовать remote
git remote rename origin old-origin

# Удалить remote
git remote remove upstream

# Изменить URL remote
git remote set-url origin git@github.com:user/repo.git

# Изменить только URL для push
git remote set-url --push origin git@github.com:user/repo.git

# Посмотреть детальную информацию о remote
git remote show origin
```

## Частые практические сценарии

### Сценарий 1: Проверить URL репозитория (забыл откуда он)

```bash
git remote -v
# или более детально
git remote show origin
```

### Сценарий 2: Добавить upstream после форка

```bash
# Вы создали форк на GitHub и склонировали его
git clone https://github.com/myname/myproject.git
cd myproject

# Добавить upstream
git remote add upstream https://github.com/original/myproject.git

# Обновиться из upstream
git fetch upstream
git rebase upstream/main
```

### Сценарий 3: Заменить HTTPS на SSH

```bash
# Текущие remote используют HTTPS
git remote -v

# Изменить на SSH (требует настройки SSH ключей)
git remote set-url origin git@github.com:username/repo.git

# Проверить
git remote -v
```

### Сценарий 4: Найти, где клонирован конкретный проект

```bash
# Ищем проект с определённым названием
find ~ -name "myproject" -type d -maxdepth 4 2>/dev/null

# Или если помните, что это git репозиторий
find ~ -type d -name ".git" -maxdepth 5 2>/dev/null -exec dirname {} \; | \
  xargs -I {} sh -c 'echo {}; git -C {} remote -v'
```

## FAQ — Часто задаваемые вопросы

### Как посмотреть список удалённых репозиториев git?

Используйте команду `git remote -v`. Она покажет имена и URL всех удалённых репозиториев текущего проекта. Команда `git remote` без флагов показывает только имена.

### Как посмотреть ветки удалённого репозитория без клонирования?

Команда `git ls-remote` позволяет увидеть все ветки и теги удалённого репозитория без клонирования:

```bash
git ls-remote --heads https://github.com/user/repo.git
```

### Как найти все git репозитории на компьютере?

На Linux/macOS используйте `find`:

```bash
find ~ -name ".git" -type d -maxdepth 5 2>/dev/null | sed 's/\/.git//'
```

На Windows используйте PowerShell:

```powershell
Get-ChildItem -Path $env:USERPROFILE -Recurse -Filter ".git" -Directory -ErrorAction SilentlyContinue
```

### Как посмотреть список своих репозиториев на GitHub?

Используйте GitHub CLI (самый удобный способ):

```bash
gh repo list
```

Или API:

```bash
curl -s -H "Authorization: token TOKEN" https://api.github.com/user/repos
```

### Что означает origin в git remote?

`origin` — это стандартное имя для удалённого репозитория, который вы клонировали. Обычно это основной репозиторий вашего проекта. Вы можете иметь несколько remote с разными именами для разных целей (origin, upstream, production и т.д.).

### Как добавить второй remote в git?

Используйте команду `git remote add`:

```bash
git remote add upstream https://github.com/original/repo.git
```

Первый аргумент — имя remote, второй — URL. После этого вы можете использовать этот remote в команде `git fetch upstream`, `git merge upstream/main` и т.д.

### Как изменить URL remote репозитория?

Используйте команду `git remote set-url`:

```bash
git remote set-url origin https://github.com/newuser/repo.git
# или на SSH
git remote set-url origin git@github.com:newuser/repo.git
```

## Заключение

Теперь вы знаете все основные способы получить список репозиториев в Git:

1. **Удалённые репозитории текущего проекта**: `git remote -v`
2. **Ветки удалённого репозитория**: `git ls-remote --heads <url>`
3. **Все локальные репозитории на диске**: `find ~ -name ".git" -type d`
4. **Репозитории на GitHub/GitLab**: `gh repo list` или API

Выбирайте нужную команду в зависимости от того, что именно вы хотите узнать. Для получения более детальной информации о работе с удалёнными репозиториями, смотрите статью про {{< relref "git-remote-v" >}} и {{< relref "git-clone" >}}.

Если вам нужно больше информации о работе с ветками на сервере, прочитайте статью про {{< relref "udalennye-vetki-git" >}}. А если вы новичок и только начинаете работать с Git, начните со статьи про {{< relref "kak-sozdat-repozitorij-github" >}}.
