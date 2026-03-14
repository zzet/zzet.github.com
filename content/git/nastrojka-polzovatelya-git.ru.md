---
title: "Настройка пользователя в Git: имя, email и параметры"
description: "Инструкция по настройке пользователя в Git: установка имени, email, локальная и глобальная конфигурация, разные профили для рабочих и личных проектов."
date: 2026-02-15
lastmod: 2026-02-15
draft: false
slug: "nastrojka-polzovatelya-git"
keywords: ["git настройка пользователя", "git config user name email", "настройка имени git", "git global config"]
tags: ["git", "beginner", "config"]
categories: ["git"]
---

Настройка пользователя в Git определяет, кто является автором коммитов. Правильная конфигурация важна: ваши коммиты будут приписаны вам на GitHub, GitLab и других платформах. Статья покажет как установить, изменить и проверить пользовательские данные — включая продвинутые сценарии с несколькими профилями.

## Основные пользовательские параметры

Два параметра обязательны и должны быть установлены перед созданием первого коммита:

```bash
# Установить имя разработчика
git config --global user.name "Иван Петров"

# Установить email разработчика
git config --global user.email "ivan@example.com"
```

Флаг `--global` означает, что эти настройки применятся ко всем репозиториям на вашем компьютере. Это рекомендуемый подход для первоначальной настройки.

Если имя или email не установлены, Git не даст создать коммит:

```
Author identity unknown

*** Please tell me who you are.

Run
  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"
```

## Три уровня конфигурации Git

Git поддерживает три уровня конфигурации, каждый перекрывает предыдущий:

**Системный уровень** (`--system`) — применяется ко всем пользователям компьютера. Хранится в `/etc/gitconfig`. Используется редко, обычно администраторами.

```bash
sudo git config --system user.name "System User"
```

**Глобальный уровень** (`--global`) — применяется ко всем репозиториям текущего пользователя. Хранится в `~/.gitconfig`.

```bash
git config --global user.name "Иван Петров"
git config --global user.email "ivan@personal.com"
```

**Локальный уровень** (без флага) — применяется только к текущему репозиторию. Хранится в `.git/config` внутри репозитория. Перекрывает глобальные настройки.

```bash
cd ~/projects/work-project
git config user.name "Иван Петров"
git config user.email "ivan@company.com"
```

## Как изменить существующие параметры

Просто повторите команду с новым значением:

```bash
# Изменить имя
git config --global user.name "Пётр Иванов"

# Изменить email
git config --global user.email "petr@example.com"
```

Новое значение перезапишет старое в файле `~/.gitconfig`.

## Проверка текущих настроек

Несколько способов проверить конфигурацию:

```bash
# Просмотр конкретного параметра
git config user.name
git config user.email

# Просмотр всех параметров с их источниками (какой файл задаёт каждый параметр)
git config --list --show-origin

# Только глобальные параметры
git config --global --list

# Только локальные параметры репозитория
git config --local --list

# Проверить, какой email использован в последнем коммите
git log -1 --format='%ae'
```

## Прямое редактирование файла конфигурации

Все параметры хранятся в текстовых файлах формата INI:

```bash
# Открыть глобальный конфиг в редакторе
git config --global --edit

# Или отредактировать файл напрямую
nano ~/.gitconfig
```

Пример содержимого `~/.gitconfig`:

```ini
[user]
    name = Иван Петров
    email = ivan@example.com
[core]
    editor = nano
    autocrlf = input
[color]
    ui = true
[alias]
    st = status
    co = checkout
    lg = log --oneline --graph --all
```

## Разные пользователи для разных проектов

Типичная ситуация: рабочие проекты должны иметь рабочий email, личные — личный. Локальная конфигурация решает это:

```bash
# Глобально — для личных проектов
git config --global user.name "Иван Петров"
git config --global user.email "ivan@gmail.com"

# Для рабочего проекта — переопределяем локально
cd ~/work/my-app
git config user.email "ivan@company.com"

# Проверить, что используется в рабочем проекте
git config user.email
# → ivan@company.com

# Коммит будет с company.com
git commit -m "Add feature"
git log -1 --format='%ae'
# → ivan@company.com
```

## Автоматический выбор профиля по папке

Git поддерживает условные include (начиная с Git 2.13). Это позволяет автоматически использовать нужный профиль в зависимости от расположения репозитория:

```ini
# ~/.gitconfig

[user]
    name = Иван Петров
    email = ivan@gmail.com

[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal
```

Создайте файл `~/.gitconfig-work`:

```ini
[user]
    email = ivan@company.com
```

Теперь все репозитории в `~/work/` автоматически используют рабочий email, а в `~/personal/` — личный. Никаких ручных действий при каждом `git clone`.

## Дополнительные параметры пользователя

```bash
# Настройка GPG-ключа для подписи коммитов
git config --global user.signingkey <GPG_KEY_ID>

# Автоматически подписывать все коммиты
git config --global commit.gpgsign true

# Требовать явную настройку identity (защита от случайного использования глобальных данных)
git config --global user.useConfigOnly true
```

## Практический пример настройки

```bash
# Первичная настройка для личных проектов
git config --global user.name "Иван Петров"
git config --global user.email "ivan.petrov@gmail.com"

# Проверка
git config --list | grep user
# user.name=Иван Петров
# user.email=ivan.petrov@gmail.com

# Переход в рабочий проект
cd ~/work/my-app
git config user.email "ivanpetrov@company.com"

# Проверка локальной конфигурации
git config --local user.email
# → ivanpetrov@company.com

# Создание коммита (будет с рабочим email)
echo "Feature X" >> features.txt
git add .
git commit -m "feat: Add feature X"

# Проверка, какой email использован в коммите
git log -1 --format='%ae'
# → ivanpetrov@company.com
```

## Часто задаваемые вопросы

**Обязательно ли использовать реальное имя?** Git не требует. Но для идентификации в команде лучше использовать реальное имя или узнаваемый ник — коллеги должны понимать, кто сделал коммит.

**Обязателен ли реальный email?** Email должен быть валидным форматом для GitHub и других сервисов. GitHub привязывает коммиты к аккаунту по email — если адрес неправильный, коммит «потеряется».

**Что произойдёт, если я установлю неправильный email?** Коммиты не свяжутся с вашим аккаунтом на GitHub/GitLab. Позже можно исправить историю с помощью `git filter-branch` или `git filter-repo`, но это сложный процесс.

**Могу ли я иметь разных пользователей для разных веток?** Нет. Пользователь определяется на уровне репозитория (или глобально), не ветки. Если нужны разные профили — используйте разные репозитории или `includeIf`.

**Как узнать, какой email я использовал для коммита?** `git log -n 1 --format='%ae'` показывает email последнего коммита. Можно также посмотреть `git show --format="%ae" HEAD`.

## Заключение

Настройка пользователя в Git проста, но важна. Правильно установленные параметры гарантируют, что ваши коммиты будут правильно атрибутированы. Особенно важно проверить email перед работой с GitHub или GitLab.

Для полного понимания настроек Git читайте [полное руководство по установке и настройке Git]({{< relref "nastrojka-git" >}}). Следующий шаг — научитесь делать [правильные коммиты]({{< relref "git-commit" >}}).
