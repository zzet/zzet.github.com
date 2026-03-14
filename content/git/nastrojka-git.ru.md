---
title: "Установка и настройка Git для Windows, macOS и Linux"
description: "Пошаговое руководство по установке Git на все ОС и первичной конфигурации. Полная подготовка к работе."
date: 2026-02-15
lastmod: 2026-02-15
draft: false
slug: "nastrojka-git"
keywords: ["git настройка", "обновить версию git", "установка git", "git config настройка", "первоначальная настройка git", "git config --global"]
tags: ["git", "beginner", "setup"]
categories: ["git"]
aliases: []
---

Установка Git — первый шаг на пути к контролю версий. Сам по себе процесс занимает несколько минут, однако после установки важно правильно настроить Git: указать имя пользователя, email и другие параметры. В этой статье разберём весь процесс для всех популярных операционных систем.

Если вы ещё не знакомы с Git и хотите сначала понять, что это такое, прочитайте статью «[Что такое Git]({{< relref "chto-takoe-git" >}})». Если Git уже установлен и вы хотите настроить SSH-доступ к GitHub, переходите к статье «[Настройка SSH для GitHub]({{< relref "nastrojka-ssh-github" >}})».

## Установка Git на Windows

### Способ 1: официальный инсталлятор (рекомендуется)

1. Откройте [git-scm.com/download/win](https://git-scm.com/download/win)
2. Скачайте 64-битный инсталлятор
3. Запустите файл `.exe`
4. В процессе установки:
   - Компонент **Git Bash** — оставьте включённым (обязательно)
   - **Git LFS** — оставьте включённым
   - **Windows Credential Manager** — оставьте включённым (упрощает авторизацию)
   - Редактор по умолчанию: выберите VS Code или Notepad++ если они установлены, иначе nano
   - Имя основной ветки: выберите `main`

### Способ 2: Chocolatey

Если у вас установлен менеджер пакетов Chocolatey:

```bash
choco install git
```

### Способ 3: Windows Package Manager (winget)

Начиная с Windows 10:

```bash
winget install --id Git.Git -e --source winget
```

Проверьте установку:
```bash
git --version
# git version 2.48.1.windows.1
```

## Установка Git на macOS

### Способ 1: Homebrew (рекомендуется)

Homebrew — стандартный менеджер пакетов для macOS. Если он ещё не установлен, откройте Terminal и выполните команду с [brew.sh](https://brew.sh).

После установки Homebrew:
```bash
brew install git
```

Это установит актуальную версию Git. Системный Git от Apple обновляется медленнее.

### Способ 2: Xcode Command Line Tools

На macOS можно вызвать установку инструментов разработки командой:
```bash
xcode-select --install
```

В открывшемся диалоге нажмите «Install». После завершения Git будет доступен в системе.

### Способ 3: официальный инсталлятор

Скачайте `.pkg` инсталлятор с [git-scm.com/download/mac](https://git-scm.com/download/mac) и следуйте инструкциям.

Проверьте установку:
```bash
git --version
# git version 2.48.0
```

## Установка Git на Linux

### Debian и Ubuntu

```bash
sudo apt update
sudo apt install git
```

### RedHat, CentOS, Fedora

```bash
# CentOS / RHEL
sudo yum install git

# Fedora 22+
sudo dnf install git
```

### Arch Linux

```bash
sudo pacman -S git
```

### Установка последней версии (Ubuntu)

Репозитории Ubuntu могут содержать устаревшую версию Git. Для установки актуальной версии:

```bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git
```

Проверьте установку:
```bash
git --version
# git version 2.48.0
```

## Первичная конфигурация Git

После установки необходимо настроить Git. Минимум — имя пользователя и email.

### Шаг 1: имя пользователя

```bash
git config --global user.name "Иван Петров"
```

Это имя будет отображаться в каждом коммите. Укажите то, которое хотите видеть в истории — полное имя или псевдоним.

### Шаг 2: email

```bash
git config --global user.email "ivan@example.com"
```

Email должен совпадать с тем, что вы используете на GitHub/GitLab — это важно для правильного привязывания коммитов к аккаунту.

### Шаг 3: проверка конфигурации

```bash
git config --list

# Вывод:
# user.name=Иван Петров
# user.email=ivan@example.com
# ...другие параметры...
```

Проверить конкретное значение:
```bash
git config user.name
# Иван Петров
```

## Уровни конфигурации Git

Git хранит настройки на трёх уровнях:

**Системный (`--system`):** настройки для всех пользователей компьютера. Файл: `/etc/gitconfig`.

**Глобальный (`--global`):** настройки для текущего пользователя, применяются ко всем репозиториям. Файл: `~/.gitconfig`.

**Локальный (без флага):** настройки только для текущего репозитория. Файл: `.git/config` внутри проекта.

Более конкретный уровень имеет приоритет. Например, если в глобальном конфиге указан один email, а в локальном другой — использоваться будет локальный.

## Дополнительные полезные настройки

### Редактор по умолчанию

Git использует текстовый редактор для написания сообщений коммитов. По умолчанию — vim, что неудобно для новичков.

```bash
# Nano (простой редактор)
git config --global core.editor "nano"

# VS Code
git config --global core.editor "code --wait"

# Vim
git config --global core.editor "vim"

# Sublime Text
git config --global core.editor "subl -n -w"
```

### Цветной вывод

Для лучшей читаемости включите цвета в терминале:

```bash
git config --global color.ui true
```

### Перенос строк (autocrlf)

Windows и Linux/macOS используют разные символы переноса строки. Чтобы избежать конфликтов в команде:

```bash
# На Windows (CRLF → LF при коммите, LF → CRLF при чекауте)
git config --global core.autocrlf true

# На macOS и Linux (оставлять как есть при чекауте)
git config --global core.autocrlf input
```

### Алиасы (сокращения команд)

Создайте короткие псевдонимы для часто используемых команд:

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"
```

После этого `git st` работает как `git status`, а `git lg` показывает красивый граф веток.

### Имя ветки по умолчанию

Современный стандарт — `main` вместо `master`:

```bash
git config --global init.defaultBranch main
```

## Файл конфигурации .gitconfig

Все глобальные настройки хранятся в файле `~/.gitconfig`. Его можно редактировать напрямую:

```bash
nano ~/.gitconfig
```

Типичное содержимое:

```ini
[user]
    name = Иван Петров
    email = ivan@example.com

[core]
    editor = nano
    autocrlf = input

[color]
    ui = true

[init]
    defaultBranch = main

[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --graph --all
```

## Обновление Git

### Windows

```bash
# Встроенный способ (Git 2.16.1+)
git update-git-for-windows
```

Или скачайте новый инсталлятор с git-scm.com.

### macOS (через Homebrew)

```bash
brew upgrade git
```

### Linux (Debian/Ubuntu)

```bash
sudo apt update
sudo apt upgrade git
```

### Проверка текущей версии

```bash
git --version
```

## Практический пример: полная настройка с нуля

```bash
# 1. Проверка установки
git --version

# 2. Базовая конфигурация
git config --global user.name "Иван Петров"
git config --global user.email "ivan@example.com"
git config --global core.editor "nano"
git config --global color.ui true
git config --global init.defaultBranch main
git config --global core.autocrlf input  # macOS/Linux

# 3. Полезные алиасы
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.lg "log --oneline --graph --all"

# 4. Проверка результата
git config --list
git config user.name
git config user.email

# 5. Просмотр файла конфигурации
cat ~/.gitconfig

# 6. Первый репозиторий
git init my_project
cd my_project
git status
```

## Часто задаваемые вопросы

**Что означает флаг `--global`?** Он применяет настройку ко всем вашим репозиториям на этом компьютере. Без флага настройка применяется только к текущему репозиторию.

**Где хранится файл `.gitconfig`?** В домашней папке: `~/.gitconfig` на Linux/macOS, `C:\Users\ИмяПользователя\.gitconfig` на Windows.

**Нужно ли указывать реальное имя и email?** Email важен, если вы будете использовать GitHub или GitLab — он связывает коммиты с вашим профилем. Имя может быть любым.

**Что будет, если я укажу неправильный email?** Это можно исправить в любой момент командой `git config --global user.email "новый@email.com"`. На будущие коммиты изменение подействует сразу.

**Нужны ли дополнительные настройки кроме user.name и user.email?** Нет, это минимально необходимый набор. Остальное — по вашему усмотрению и привычкам.

## Заключение

Установка и настройка Git занимают несколько минут. После выполнения базовых шагов вы готовы к работе с контролем версий. Не откладывайте настройку SSH-ключей — это сделает работу с GitHub значительно удобнее, и об этом рассказывает следующая статья «[Настройка SSH для GitHub]({{< relref "nastrojka-ssh-github" >}})».

Правильно указанный email важен для того, чтобы ваши коммиты корректно отображались в GitHub и привязывались к вашему профилю.
