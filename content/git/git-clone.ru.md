---
title: "git clone: как скачать репозиторий с GitHub или GitLab"
description: "Полное руководство по git clone. HTTPS vs SSH, поверхностное клонирование, примеры команд, что делать после clone."
date: 2026-01-07
lastmod: 2026-01-07
draft: false
slug: "git-clone"
keywords: ["ssh git clone", "git clone что это", "как скачать репозиторий git", "скачать код с github"]
tags: ["git", "remote", "beginner"]
categories: ["git"]
aliases: []
---

`git clone` — первая команда, которую выполняет любой разработчик при присоединении к проекту. Она создаёт полную копию удалённого репозитория на вашем компьютере, включая всю историю коммитов, все ветки и теги. В этой статье разберём все способы клонирования и что делать после.

## Что происходит при git clone

Когда вы клонируете репозиторий, Git:

1. Создаёт новую папку с именем репозитория
2. Инициализирует в ней Git (папка `.git`)
3. Скачивает все коммиты, ветки и теги
4. Настраивает `origin` на исходный URL
5. Создаёт локальную ветку, отслеживающую основную удалённую ветку
6. Переключается на основную ветку (обычно `main` или `master`)

После клонирования вы получаете полноценный локальный репозиторий.

## Базовое клонирование

```bash
git clone https://github.com/username/repository.git
# Создаст папку repository/ в текущей директории

cd repository
git log --oneline -5  # История коммитов уже здесь
git branch -a         # Все ветки
```

### Клонирование в конкретную папку

```bash
git clone https://github.com/username/repository.git my-project
# Создаст папку my-project вместо repository
```

### Клонирование в текущую папку

```bash
mkdir my-project
cd my-project
git clone https://github.com/username/repository.git .
# Точка означает "текущую папку"
```

## HTTPS vs SSH: что выбрать

Есть два протокола для клонирования:

### HTTPS (проще для начала)

```bash
git clone https://github.com/username/repository.git
```

При push потребует логин/пароль или токен. Удобно для одноразового использования или если не настроены SSH-ключи.

### SSH (рекомендуется для постоянной работы)

```bash
git clone git@github.com:username/repository.git
```

Требует настроенных SSH-ключей, но зато не спрашивает пароль при каждом push. Стандарт для профессиональной работы.

URL для SSH начинается с `git@`, URL для HTTPS — с `https://`. На GitHub нажмите кнопку **Code** и выберите **SSH** или **HTTPS**.

Подробнее о настройке SSH — в статье «[Настройка SSH для GitHub]({{< relref "nastrojka-ssh-github" >}})».

### Изменение протокола в существующем репозитории

Если клонировали через HTTPS, а хотите переключиться на SSH:

```bash
git remote set-url origin git@github.com:username/repository.git
git remote -v  # Проверить
```

## Клонирование конкретной ветки

По умолчанию клонируется основная ветка. Для другой ветки:

```bash
git clone -b develop https://github.com/username/repository.git

# Или с флагом --branch
git clone --branch develop https://github.com/username/repository.git
```

## Поверхностное клонирование (shallow clone)

Большие репозитории с долгой историей могут занимать много места. Поверхностное клонирование загружает только последние N коммитов:

```bash
# Только последний коммит
git clone --depth 1 https://github.com/username/large-repo.git

# Последние 10 коммитов
git clone --depth 10 https://github.com/username/large-repo.git

# Конкретная ветка + shallow
git clone --depth 1 --branch main https://github.com/username/large-repo.git
```

Полезно для CI/CD и случаев, когда история не нужна.

### Преобразование shallow в полный репозиторий

```bash
git fetch --unshallow
```

## Клонирование только одной ветки

```bash
git clone --single-branch -b main https://github.com/username/repository.git
```

Скачивает только одну ветку вместо всех. Экономит место и время.

## Клонирование с подмодулями

Если репозиторий содержит подмодули (submodules):

```bash
# Клонировать вместе с подмодулями
git clone --recursive https://github.com/username/repository.git

# Если забыли --recursive:
git submodule update --init --recursive
```

## Что находится внутри клонированного репозитория

```bash
cd repository
ls -la

# .git/         — служебная папка Git (история, конфиги)
# .gitignore    — правила игнорирования файлов
# README.md     — документация
# src/          — код проекта
# ...

git remote -v
# origin  git@github.com:username/repository.git (fetch)
# origin  git@github.com:username/repository.git (push)

git branch -a
# * main                    ← текущая локальная ветка
#   remotes/origin/main
#   remotes/origin/develop
#   remotes/origin/feature/xyz
```

## Первые шаги после clone

```bash
# 1. Войти в папку
cd repository

# 2. Посмотреть ветки
git branch -a

# 3. Посмотреть историю
git log --oneline -10

# 4. Установить зависимости (в зависимости от проекта)
npm install          # Node.js
pip install -r requirements.txt  # Python
bundle install       # Ruby

# 5. Создать свою ветку для работы
git switch -c feature/my-task

# 6. Начать работу
```

## Практические примеры

```bash
# Клонировать open-source проект
git clone https://github.com/torvalds/linux.git

# Это займёт много времени! Лучше shallow:
git clone --depth 1 https://github.com/torvalds/linux.git

# Клонировать приватный репозиторий по SSH
git clone git@github.com:mycompany/private-project.git

# Клонировать конкретную ветку
git clone -b v2.1-stable https://github.com/username/repo.git

# Зеркалирование (резервная копия)
git clone --mirror https://github.com/username/repository.git backup.git
```

## Клонирование без checkout

```bash
git clone --no-checkout https://github.com/username/repository.git
# Создаёт .git, но не разворачивает файлы
# Полезно для настройки sparse checkout
```

## Часто задаваемые вопросы

**Какой протокол использовать: HTTPS или SSH?** Для случайного использования — HTTPS (проще). Для постоянной работы — SSH (не нужно вводить пароль). Настройте SSH-ключи один раз и забудьте о паролях.

**Как клонировать только одну ветку?** `git clone --single-branch -b имя-ветки URL`. Это уменьшает объём скачиваемых данных.

**Что такое поверхностное клонирование?** `git clone --depth N` скачивает только последние N коммитов. Экономит место и время, но теряет старую историю. Полезно для CI/CD.

**Где найти URL для клонирования?** На GitHub: зелёная кнопка Code → выбрать HTTPS или SSH → скопировать URL. На GitLab: синяя кнопка Clone → выбрать протокол.

**Что такое `origin` после клонирования?** Автоматически созданный псевдоним для URL исходного репозитория. `git push origin main` отправит изменения обратно туда, откуда вы клонировали.

**Как обновить клонированный репозиторий?** `git pull origin main` для получения новых коммитов из основной ветки.

## Заключение

`git clone` — простая, но важная команда. Для постоянной работы настройте SSH-ключи и используйте SSH URL — это удобнее и безопаснее. Для больших репозиториев в CI/CD используйте `--depth 1` для экономии времени.

После клонирования: проверьте ветки, установите зависимости, создайте свою ветку и начинайте работу. Все изменения отправляйте через `git push`, обновляйте через `git pull`.
