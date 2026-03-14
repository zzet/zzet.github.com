---
title: "Файл .zshenv: переменные окружения для разработчика"
description: "Что такое .zshenv и как его настроить. Переменные окружения PATH, GIT_EDITOR, NODE_PATH. Отличие от .zshrc, типичные примеры настройки для разработки."
date: 2026-03-14
lastmod: 2026-03-14
draft: false
slug: "zshenv"
keywords: ["zshenv", "что такое zshenv", "zshenv переменные окружения git", "zshenv файл", ".zshenv настройка", "переменные окружения zsh"]
tags: ["git", "terminal", "intermediate"]
categories: ["git"]
---

`.zshenv` — файл конфигурации Zsh, который загружается при каждом запуске оболочки, включая неинтерактивные и non-login сессии. Это лучшее место для переменных окружения, которые должны быть доступны везде — в терминале, в скриптах и в GUI-приложениях.

## Что такое .zshenv

Zsh при запуске читает несколько файлов конфигурации в определённом порядке:

```
Порядок загрузки (от первого к последнему):
1. /etc/zshenv        — системный (для всех пользователей)
2. ~/.zshenv          — пользовательский ← здесь переменные окружения
3. /etc/zprofile      — системный login-shell
4. ~/.zprofile        — пользовательский login-shell
5. /etc/zshrc         — системный интерактивный
6. ~/.zshrc           — пользовательский интерактивный
7. /etc/zlogin        — системный login-shell (финальный)
8. ~/.zlogin          — пользовательский login-shell (финальный)
```

`.zshenv` загружается **всегда** — это делает его идеальным местом для переменных окружения.

## Отличие от .zshrc

```
.zshenv:
- Загружается при каждом запуске Zsh (интерактивном и нет)
- Предназначен для переменных окружения (export VAR=value)
- Также загружается в скриптах и cron-задачах
- НЕ для псевдонимов и функций

.zshrc:
- Загружается только для интерактивных оболочек
- Для псевдонимов (alias), prompt, функций
- Для настройки plagers (oh-my-zsh, starship)
- НЕ загружается в скриптах
```

**Правило**: если переменная нужна в скриптах — в `.zshenv`. Если только в интерактивном терминале — в `.zshrc`.

## Типичное содержимое .zshenv

```bash
# ~/.zshenv

# PATH — основная переменная
export PATH="$HOME/.local/bin:$HOME/bin:$PATH"

# Homebrew (macOS)
export PATH="/opt/homebrew/bin:/opt/homebrew/sbin:$PATH"

# Переменные для Git
export GIT_EDITOR="vim"           # редактор для коммитов
export GIT_AUTHOR_NAME="Иван Иванов"
export GIT_AUTHOR_EMAIL="ivan@example.com"

# Node.js
export NVM_DIR="$HOME/.nvm"
export NODE_PATH="$HOME/.nvm/versions/node/v20.0.0/lib/node_modules"

# Python
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"

# Java
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk"
export PATH="$JAVA_HOME/bin:$PATH"

# Общие переменные
export EDITOR="vim"
export VISUAL="code --wait"
export LANG="en_US.UTF-8"
export LC_ALL="en_US.UTF-8"
```

## Переменные Git в .zshenv

Git использует переменные окружения для настройки поведения:

```bash
# ~/.zshenv — переменные для Git

# Редактор сообщений коммитов
export GIT_EDITOR="vim"
# или VS Code:
export GIT_EDITOR="code --wait"
# или nano:
export GIT_EDITOR="nano"

# Альтернатива: через git config (приоритет выше env)
# git config --global core.editor "code --wait"

# Пейджер для вывода (git log, git diff)
export GIT_PAGER="less -FX"
# или отключить:
export GIT_PAGER=""

# SSH ключ для Git операций
export GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519_work"

# Цвет в выводе
export GIT_TERMINAL_PROMPT=1

# Для отладки (показывает curl запросы)
# export GIT_TRACE=1
# export GIT_CURL_VERBOSE=1
```

## PATH для инструментов разработки

```bash
# ~/.zshenv — добавление путей к инструментам

# Homebrew (macOS arm64)
export PATH="/opt/homebrew/bin:$PATH"

# nvm (Node.js version manager)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && source "$NVM_DIR/nvm.sh"

# rbenv (Ruby version manager)
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init - zsh)"

# Go
export GOPATH="$HOME/go"
export PATH="$GOPATH/bin:$PATH"

# Rust (cargo)
export PATH="$HOME/.cargo/bin:$PATH"

# Пользовательские скрипты
export PATH="$HOME/scripts:$PATH"
```

## Создание и редактирование .zshenv

```bash
# Создать файл если нет
touch ~/.zshenv

# Открыть в редакторе
nano ~/.zshenv
# или
vim ~/.zshenv
# или
code ~/.zshenv

# Применить изменения немедленно (без перезапуска терминала)
source ~/.zshenv

# Проверить что переменная установлена
echo $GIT_EDITOR
echo $PATH
```

## Проверка и отладка

```bash
# Показать все переменные окружения
env | sort

# Найти конкретную переменную
env | grep GIT

# Проверить где определена переменная
echo $PATH
which git
which node

# Трассировка загрузки Zsh (отладка)
zsh -xvls 2>&1 | head -50

# Проверить что .zshenv загружается в скриптах
zsh -c 'echo $GIT_EDITOR'  # должен показать значение
bash -c 'echo $GIT_EDITOR' # bash не читает .zshenv
```

## Работа на macOS

На macOS переменные окружения для GUI-приложений (VS Code, Finder) нужно устанавливать иначе:

```bash
# ~/.zshenv читается только терминальными приложениями
# Для GUI-приложений на macOS:

# Способ 1: launchctl (устаревший, но работает)
launchctl setenv GIT_EDITOR vim

# Способ 2: ~/.config/environment.plist
# (специфично для macOS)

# На практике: VS Code и JetBrains читают PATH из терминала
# если запущены из терминала:
code .           # запуск из терминала — унаследует PATH
open -a "VS Code" . # запуск из Finder — может не унаследовать
```

## Часто задаваемые вопросы

**Почему PATH не работает в GUI-приложениях?** GUI-приложения на macOS не запускают login shell, поэтому `.zshenv` и `.zshrc` не читаются. Решение: запускать IDE из терминала (`code .`).

**Что делать если в .zshenv ошибка?** Zsh не запустится. Исправить через другую оболочку: `bash ~/.zshenv` покажет ошибку. Или открыть файл через Finder/другой редактор.

**Можно ли хранить секреты в .zshenv?** Нежелательно — файл хранится в открытом виде. Для API ключей лучше использовать менеджер секретов (1Password CLI, Vault) или `.env` файлы (не попадающие в Git).

**Что приоритетнее: .zshenv или git config?** Git-специфичные переменные (GIT_EDITOR) в git config имеют приоритет над переменными окружения. Для PATH и системных переменных — только `.zshenv` работает.

**Как проверить что .zshenv загружается?** Добавьте `echo ".zshenv loaded"` в файл и откройте новый терминал. Если видите сообщение — файл загружается.

## Заключение

`.zshenv` — правильное место для переменных окружения (PATH, GIT_EDITOR, JAVA_HOME) в Zsh. В отличие от `.zshrc`, загружается при любом запуске оболочки — в интерактивных сессиях, скриптах и cron-задачах. Изменения применяются после `source ~/.zshenv`. Для настройки Git редактора используйте `GIT_EDITOR` в `.zshenv` или `git config --global core.editor`. Подробнее о настройке Git — [настройка Git]({{< relref "nastrojka-git" >}}).
