---
title: "git init vs git clone: когда что использовать"
description: "Разница между git init и git clone. git init создаёт новый репозиторий, git clone копирует существующий. Примеры использования, сравнение, типичные случаи."
date: 2026-01-17
lastmod: 2026-01-17
draft: false
slug: "git-init-vs-git-clone"
keywords: ["git init vs git clone", "git init или git clone", "разница git init git clone", "когда использовать git init", "git init новый проект", "git clone существующий проект"]
tags: ["git", "beginner"]
categories: ["git"]
---

`git init` и `git clone` — две команды для начала работы с Git репозиторием. `git init` создаёт новый пустой репозиторий. `git clone` копирует существующий репозиторий с историей. Выбор зависит от того, начинаете ли вы с нуля или подключаетесь к уже существующему проекту.

## git init: создание нового репозитория

`git init` инициализирует Git в текущей папке — создаёт папку `.git` со всеми необходимыми файлами:

```bash
# Создать новый проект с Git
mkdir my-project
cd my-project
git init
# Initialized empty Git repository in /home/user/my-project/.git/

# Или инициализировать в новой папке
git init my-project
# Initialized empty Git repository in /home/user/my-project/.git/

# Создать bare репозиторий (для сервера)
git init --bare my-project.git
```

После `git init` создаётся папка `.git`, но нет ни одного коммита, ни одной ветки, ни одного remote.

**Типичный сценарий использования:**

```bash
# Начать новый проект
mkdir blog-api
cd blog-api
git init

# Добавить файлы
echo "# Blog API" > README.md
git add README.md
git commit -m "Initial commit"

# Подключить к GitHub (создайте репозиторий на GitHub сначала)
git remote add origin git@github.com:username/blog-api.git
git branch -M main
git push -u origin main
```

## git clone: копирование существующего репозитория

`git clone` создаёт полную копию репозитория — со всей историей коммитов, всеми ветками и тегами:

```bash
# Клонировать с GitHub
git clone https://github.com/user/project.git

# Клонировать в конкретную папку
git clone https://github.com/user/project.git my-local-name

# Клонировать только одну ветку (мелкое клонирование)
git clone --single-branch --branch main https://github.com/user/project.git

# Клонировать с ограниченной историей (ускоряет клонирование)
git clone --depth 1 https://github.com/user/project.git

# Клонировать с SSH
git clone git@github.com:user/project.git
```

После `git clone`:
- Вся история проекта скопирована
- Автоматически настроен remote `origin`
- Текущая ветка установлена на ветку по умолчанию

## Сравнение: init vs clone

```
Характеристика     git init              git clone
Назначение         Новый репозиторий     Копия существующего
История коммитов   Пустая                Полная копия
Remote origin      Нет                   Настроен автоматически
Ветки              Нет (до 1 коммита)    Все удалённые ветки
Файлы проекта      Только что есть       Все файлы из репозитория
Когда использовать Новый проект          Присоединение к проекту
```

## Когда использовать git init

```bash
# 1. Начинаете новый проект с нуля
mkdir new-project && cd new-project
git init

# 2. Хотите добавить Git к существующему проекту (без Git)
cd existing-project-without-git
git init
git add .
git commit -m "Add existing code to git"

# 3. Создаёте сервер/центральный репозиторий
git init --bare /srv/git/myapp.git

# 4. Создаёте локальное зеркало для тестирования
git init test-repo
cd test-repo
git remote add origin /path/to/real/repo
git fetch --all
```

## Когда использовать git clone

```bash
# 1. Подключаетесь к существующему командному проекту
git clone git@github.com:company/main-app.git

# 2. Вносите вклад в open source проект
git clone https://github.com/popular/library.git

# 3. Делаете локальную копию своего GitHub репозитория
git clone git@github.com:yourusername/your-project.git

# 4. Создаёте локальную копию для эксперимента
git clone /path/to/local/repo experiment-copy

# 5. CI/CD: получить код на сервере
git clone --depth 1 https://github.com/user/project.git
```

## git init после git clone: не нужно

Частая ошибка — запуск `git init` в папке, которая уже является Git репозиторием:

```bash
# Неправильно: init в клонированном репозитории
git clone https://github.com/user/project.git
cd project
git init  # Ошибка! Уже является репозиторием
# Reinitialized existing Git repository in /home/user/project/.git/

# git init в существующем репозитории — безвредно, но лишнее
# История и настройки сохраняются
```

## Воссоздание связи: init + remote

Иногда нужно инициализировать новый репозиторий и связать его с существующим удалённым:

```bash
# Случай: потеряли локальный репозиторий, нужно восстановить
# Вместо git clone используем init + remote + pull

# Шаг 1: инициализировать
cd new-location
git init

# Шаг 2: добавить remote
git remote add origin git@github.com:user/project.git

# Шаг 3: получить данные
git fetch origin

# Шаг 4: переключиться на ветку
git checkout -b main origin/main
# Или:
git switch -c main origin/main

# Проще: просто использовать git clone
git clone git@github.com:user/project.git new-location
```

## Параметры git clone

```bash
# --depth N: поверхностное клонирование (только N последних коммитов)
# Быстрее, меньше места, но нет полной истории
git clone --depth 1 https://github.com/user/large-project.git

# --branch: клонировать конкретную ветку
git clone --branch develop https://github.com/user/project.git

# --single-branch: только одна ветка (не скачивает все)
git clone --single-branch --branch main https://github.com/user/project.git

# --bare: клонировать как bare репозиторий (без рабочей директории)
git clone --bare https://github.com/user/project.git project.git

# --mirror: полное зеркало (все refs включая pull requests)
git clone --mirror https://github.com/user/project.git

# --recurse-submodules: также клонировать подмодули
git clone --recurse-submodules https://github.com/user/project.git
```

## Параметры git init

```bash
# Инициализировать с нужным именем ветки по умолчанию
git init --initial-branch=main
# Или через config:
git config --global init.defaultBranch main

# Bare репозиторий
git init --bare

# С templatedir (шаблоны хуков и файлов)
git init --template=/path/to/templates

# С shared (для серверов, группой пользователей)
git init --shared=group
```

## Часто задаваемые вопросы

**Можно ли использовать git init для подключения к существующему репозиторию?** Технически да: `git init`, потом `git remote add origin URL`, `git fetch`, `git checkout main`. Но проще сразу использовать `git clone`.

**git clone копирует все ветки?** `git clone` скачивает все объекты и ссылки, но автоматически переключается только на ветку по умолчанию. Все удалённые ветки доступны как `origin/branch-name`.

**Как ускорить git clone большого репозитория?** Используйте `--depth 1` для поверхностного клонирования или `--single-branch --branch main` для скачивания только нужной ветки.

**git init создаёт пустой коммит?** Нет. `git init` только создаёт `.git` папку. Репозиторий не содержит ни одного коммита до первого `git commit`.

**Что делает git clone под капотом?** `git clone` выполняет: `git init` → `git remote add origin URL` → `git fetch` → `git checkout` на ветку по умолчанию.

## Заключение

`git init` — для нового проекта или добавления Git к существующему проекту без Git. `git clone` — для копирования уже существующего репозитория, подключения к командному проекту или open source. В большинстве случаев работы в команде вы будете использовать `git clone`. `git init` нужен когда начинаете с чистого листа.

## По теме

- [git init]({{< relref "git-init" >}})
- [git clone]({{< relref "git-clone" >}})
- [Что такое репозиторий]({{< relref "chto-takoe-repozitoriy" >}})
- [Создать локальный репозиторий]({{< relref "sozdat-lokalnyj-repozitorij" >}})
