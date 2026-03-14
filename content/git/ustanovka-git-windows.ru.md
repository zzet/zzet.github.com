---
title: "Установка Git на Windows: скачать, настроить и начать работу"
description: "Пошаговое руководство по установке Git на Windows. Официальный установщик git-scm.com, настройка Git Bash, первоначальная конфигурация user.name и user.email."
date: 2026-01-02
lastmod: 2026-01-02
draft: false
slug: "ustanovka-git-windows"
keywords: ["установка git windows", "скачать git", "git для windows", "git install windows", "git bash скачать", "git download windows"]
tags: ["git", "windows", "beginner"]
categories: ["git"]
---

Git — обязательный инструмент любого разработчика. На Windows установка немного отличается от Linux и macOS: нужно выбрать правильные параметры в установщике и настроить окружение. Весь процесс занимает около пяти минут.

## Где скачать Git для Windows

Официальный источник — **git-scm.com/download/win**. Никаких сторонних сайтов, только официальный дистрибутив.

```
Шаги:
1. Открыть git-scm.com/download/win
2. Загрузка начнётся автоматически для 64-bit Windows
3. Запустить скачанный .exe файл
```

Текущая версия всегда отображается на странице загрузки. На момент написания статьи — Git 2.44+.

## Пошаговая установка

### Шаг 1: Лицензионное соглашение
Нажать **Next**.

### Шаг 2: Выбор компонентов
Оставить значения по умолчанию. Убедитесь что включены:
- **Git Bash Here** — контекстное меню в Проводнике
- **Git GUI Here** — опционально
- **Associate .git* configuration files** — полезно

### Шаг 3: Выбор редактора по умолчанию
По умолчанию предлагается **Vim** — это может сбить с толку новичков. Рекомендуется выбрать **Visual Studio Code** или **Notepad++**:

```
Выпадающий список "Choosing the default editor used by Git"
→ выбрать "Use Visual Studio Code as Git's default editor"
```

Если VS Code не установлен — выберите Notepad++ или Nano.

### Шаг 4: Имя ветки по умолчанию
Выбрать **Override the default branch name** и вписать `main`. Это современное соглашение.

### Шаг 5: Настройка PATH (важно!)
Выбрать **Git from the command line and also from 3rd-party software**. Это добавляет Git в системный PATH — `git` будет работать в CMD, PowerShell и любом терминале.

```
❌ "Use Git from Git Bash only" — Git не будет доступен вне Bash
✅ "Git from the command line and also from 3rd-party software" — рекомендуется
❌ "Use Git and optional Unix tools from Windows Command Prompt" — может конфликтовать с Windows утилитами
```

### Шаг 6: SSH-клиент
Выбрать **Use bundled OpenSSH** — встроенный SSH-клиент Git. Не требует отдельной установки PuTTY.

### Шаг 7: Настройка HTTPS
Выбрать **Use the OpenSSL library** — стандартный вариант.

### Шаг 8: Переносы строк (line endings)
Это важный параметр при работе в команде:

```
✅ "Checkout Windows-style, commit Unix-style line endings" — рекомендуется
   git config core.autocrlf true
   При checkout: LF → CRLF (Windows)
   При commit: CRLF → LF (Unix)
```

### Шаг 9: Эмулятор терминала
Выбрать **Use MinTTY** — более удобный терминал, поддерживает цвета и Unicode.

### Шаг 10: git pull behaviour
Оставить **Default (fast-forward or merge)**.

### Шаги 11–13
Оставить значения по умолчанию. Нажимать **Next** до **Install**.

## Проверка установки

После завершения откройте **Git Bash** (найти в меню Пуск или правый клик в папке → Git Bash Here):

```bash
git --version
# git version 2.44.0.windows.1

where git
# C:\Program Files\Git\cmd\git.exe
```

Также работает в PowerShell и CMD:

```powershell
# PowerShell
git --version
# git version 2.44.0.windows.1
```

## Первоначальная настройка

После установки **обязательно** настроить имя и email — они будут отображаться в каждом коммите:

```bash
git config --global user.name "Ваше Имя"
git config --global user.email "your@email.com"
```

Рекомендуемые дополнительные настройки:

```bash
# Редактор для коммит-сообщений
git config --global core.editor "code --wait"   # VS Code
# git config --global core.editor "notepad++"  # Notepad++

# Имя ветки по умолчанию
git config --global init.defaultBranch main

# Цвета в выводе
git config --global color.ui auto

# Проверить все настройки
git config --global --list
```

## Git Bash: что это и зачем

**Git Bash** — эмулятор терминала для Windows, включённый в установку Git. Он предоставляет Unix-команды: `ls`, `mkdir`, `rm`, `ssh`, `curl`. Большинство документации по Git написана с расчётом на Unix-терминал, поэтому Git Bash — рекомендуемый инструмент для работы на Windows.

```bash
# Полезные команды в Git Bash
pwd           # текущая директория
ls -la        # список файлов
cd /c/Users   # переход в C:\Users
mkdir project # создать папку
```

## Git Bash vs PowerShell vs CMD

```
Git Bash:    рекомендуется — Unix синтаксис как в документации
PowerShell:  работает хорошо, но синтаксис отличается (ls, rm ведут себя иначе)
CMD:         работает, но устаревший интерфейс
Windows Terminal: лучший хост для любого из выше
```

Если вы используете PowerShell — учитывайте что некоторые псевдонимы (`ls`, `rm`) переопределены PowerShell'ом и ведут себя иначе чем в Unix.

## Первый репозиторий на Windows

```bash
# Создать папку для проекта
mkdir my-first-project
cd my-first-project

# Инициализировать репозиторий
git init
# Initialized empty Git repository in C:/Users/user/my-first-project/.git/

# Создать файл
echo "# My Project" > README.md

# Добавить и закоммитить
git add README.md
git commit -m "Initial commit"

# Проверить историю
git log --oneline
# abc1234 Initial commit
```

## Обновление Git на Windows

```bash
# Git 2.16.1+ умеет обновляться сам
git update-git-for-windows
# Git for Windows 2.44.0 (2024-02-23) → latest

# Или просто скачать новый установщик с git-scm.com
```

## Часто задаваемые вопросы

**Нужны ли права администратора для установки?** Да, для установки в `C:\Program Files\Git`. Если нет прав — скачайте portable версию с git-scm.com/download/win (не требует установки).

**Что выбрать: Git Bash или PowerShell?** Git Bash — для тех кто следует документации и туториалам (большинство примеров написаны в Unix синтаксисе). PowerShell — если вы уже в нём работаете и знаете его особенности.

**Git установлен, но не находится в VS Code?** VS Code обнаруживает Git автоматически при следующем запуске. Если не находит — `File → Preferences → Settings → git.path` → указать путь к `git.exe`.

**Как проверить что Git добавлен в PATH?** Открыть CMD или PowerShell и написать `git --version`. Если команда найдена — PATH настроен правильно.

**Можно ли установить несколько версий Git?** Не рекомендуется — вызовет конфликты в PATH. Обновляйте через `git update-git-for-windows`.

## Заключение

Установка Git на Windows: скачать с git-scm.com, выбрать «Git from command line», настроить редактор (не Vim), установить имя ветки `main`. После установки выполнить `git config --global user.name` и `git config --global user.email`. Следующий шаг — [настройка SSH для GitHub]({{< relref "nastrojka-ssh-github" >}}) или [первый репозиторий]({{< relref "git-init" >}}).
