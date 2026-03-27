---
title: "Как создать репозиторий на GitHub: пошаговое руководство для новичков"
description: "Полная инструкция по созданию репозитория на GitHub. От регистрации до первого push. README, .gitignore, лицензии, примеры команд."
date: 2026-02-09
lastmod: 2026-02-09
draft: false
slug: "kak-sozdat-repozitorij-github"
keywords: ["как создать репозиторий git для проекта", "репозиторий на гитхабе", "создание репозитория github", "создать репозиторий github", "новый репозиторий github", "github new repository"]
tags: ["git", "github", "beginner"]
categories: ["git"]
aliases: []
---

GitHub — самая популярная платформа для хостинга Git-репозиториев. Здесь хранятся проекты от небольших учебных заданий до ядра Linux. Создать репозиторий можно за несколько минут. В этой статье разберём весь процесс — от регистрации до первого коммита.

## Шаг 1: создание аккаунта на GitHub

Если у вас ещё нет аккаунта:

1. Откройте [github.com](https://github.com)
2. Нажмите **Sign up**
3. Введите email, придумайте имя пользователя и пароль
4. Подтвердите email
5. Выберите план (Free достаточно для большинства задач)

Имя пользователя будет частью URL ваших репозиториев: `github.com/username/repository`.

## Шаг 2: создание нового репозитория

После входа в аккаунт:

**Способ 1:** нажмите зелёную кнопку **New** на главной странице или на странице `/repositories`.

**Способ 2:** нажмите **+** в правом верхнем углу → **New repository**.

**Способ 3:** перейдите напрямую на `github.com/new`.

## Шаг 3: заполнение информации о репозитории

### Repository name (обязательно)

Имя репозитория — часть URL. Правила:
- Только буквы, цифры, дефисы и подчёркивания
- Без пробелов
- Лучше в нижнем регистре через дефис: `my-web-project`

```
Хорошо: my-project, webapp, learn-python
Плохо:  My Project, project@2024, проект
```

### Description (опционально)

Краткое описание проекта. Отображается на странице репозитория. Полезно для публичных проектов.

### Public vs Private

**Public** — репозиторий виден всем. Хорошо для open-source, учебных проектов, портфолио.

**Private** — виден только вам и приглашённым участникам. Для коммерческих проектов и личного кода.

### Инициализация репозитория

GitHub предлагает три опции при создании:

**Add a README file** — создаст `README.md` с именем проекта. Рекомендуется для новых проектов. README — это «лицо» репозитория.

**Add .gitignore** — выберите шаблон по типу проекта (Python, Node.js, Java и т.д.). Git будет игнорировать файлы согласно этому шаблону. Подробнее о `.gitignore` — в отдельной статье.

**Choose a license** — лицензия определяет, как другие могут использовать ваш код:
- **MIT** — максимально свободная, можно использовать для любых целей
- **Apache 2.0** — похожа на MIT, но с условием сохранения патентных прав
- **GPL v3** — производные работы должны быть тоже открытыми (copyleft)
- Без лицензии — по умолчанию все права защищены

### Шаг 4: создание репозитория

Нажмите **Create repository**. GitHub создаст репозиторий и откроет его страницу.

## Шаг 5: клонирование репозитория

Скопируйте URL репозитория:
- Нажмите зелёную кнопку **Code**
- Выберите **SSH** (если настроены ключи) или **HTTPS**
- Скопируйте URL

```bash
# Клонировать по SSH (рекомендуется)
git clone git@github.com:username/my-project.git

# Клонировать по HTTPS
git clone https://github.com/username/my-project.git

# Перейти в папку
cd my-project
```

## Шаг 6: первый коммит и push

### Если инициализировали с README

```bash
# Репозиторий уже содержит файлы
git log --oneline
# a1b2c3d Initial commit

# Создать свою ветку для разработки
git switch -c develop

# Добавить файлы проекта
echo "console.log('Hello GitHub!');" > index.js
git add index.js
git commit -m "feat: add initial JavaScript"

# Отправить
git push -u origin develop
```

### Если создали пустой репозиторий

GitHub покажет инструкции. Вот стандартный путь:

```bash
# Инициализировать локально
git init
echo "# My Project" > README.md
git add README.md
git commit -m "Initial commit"

# Связать с GitHub
git remote add origin git@github.com:username/my-project.git
git branch -M main
git push -u origin main
```

### Если у вас уже есть локальный проект

```bash
cd /path/to/existing/project

# Если Git ещё не инициализирован
git init

# Добавить все файлы и создать первый коммит
git add .
git commit -m "Initial commit: add existing project"

# Связать с GitHub и запушить
git remote add origin git@github.com:username/my-project.git
git branch -M main
git push -u origin main
```

## Управление репозиторием

### Защита основной ветки

Чтобы в `main` нельзя было пушить напрямую без Pull Request:

1. Settings → Branches → Add rule
2. Введите `main` в Branch name pattern
3. Включите: Require pull request before merging
4. Нажмите Create

### Добавление участников

1. Settings → Collaborators
2. Add people → введите имя пользователя GitHub
3. Выберите роль (Read, Write, Admin)

### Переименование репозитория

1. Settings → General
2. Repository name → введите новое имя
3. Rename

Старые URL будут автоматически перенаправляться на новые.

## Создание .gitignore вручную

Если создавали репозиторий без .gitignore:

```bash
# Для Node.js проекта
cat > .gitignore << 'EOF'
node_modules/
npm-debug.log
.env
.env.local
dist/
build/
.DS_Store
EOF

# Для Python
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
venv/
.env
*.egg-info/
dist/
build/
EOF

git add .gitignore
git commit -m "chore: add .gitignore"
git push
```

## Практический пример: создание учебного проекта

```bash
# 1. Создайте репозиторий на GitHub (через веб-интерфейс)
# Имя: learn-git, Public, Add README

# 2. Клонируйте
git clone git@github.com:username/learn-git.git
cd learn-git

# 3. Посмотрите что есть
ls
cat README.md
git log --oneline

# 4. Создайте ветку для задачи
git switch -c feature/add-examples

# 5. Добавьте файлы
echo "print('Hello, Git!')" > hello.py
git add hello.py
git commit -m "feat: add Python hello world example"

# 6. Запушьте ветку
git push -u origin feature/add-examples

# 7. Создайте Pull Request на GitHub
# (через веб-интерфейс или GitHub CLI: gh pr create)

# 8. После merge — обновите main
git switch main
git pull origin main
git branch -d feature/add-examples
```

## Часто задаваемые вопросы

**Нужно ли создавать README при создании репозитория?** Для публичных проектов — да. README — первое, что видят посетители. Для приватных учебных проектов — по желанию. Если вы планируете сразу пушить существующий код — можно создать пустой репозиторий и добавить README потом.

**Какую лицензию выбрать?** Для большинства open-source проектов — MIT. Она простая и позволяет максимально свободно использовать код. Для корпоративных проектов проконсультируйтесь с юристом.

**Чем HTTPS отличается от SSH?** HTTPS проще настроить, но требует ввода учётных данных (или токена). SSH работает по ключам — один раз настроить и больше не вводить пароль. Для регулярной работы SSH удобнее.

**Как пригласить других в репозиторий?** Settings → Collaborators → Add people. Введите имя пользователя GitHub. Для организации используйте Teams.

**Можно ли изменить название репозитория?** Да, в Settings → General. Старые ссылки будут перенаправляться, но лучше обновить remote URL локально: `git remote set-url origin новый-URL`.

**Как удалить репозиторий?** Settings → Danger Zone → Delete this repository. Нужно подтвердить, введя имя репозитория. Это необратимо.

## Заключение

Создание репозитория на GitHub — простой процесс из нескольких кликов. Главные решения: имя, публичный или приватный, нужна ли лицензия. После создания клонируйте репозиторий, добавьте код, создайте коммит и запушьте.

GitHub — это не только хранилище кода. Это Pull Requests для code review, Issues для отслеживания задач, GitHub Actions для CI/CD и Pages для публикации сайтов. Изучайте эти возможности по мере необходимости.

Следующий шаг — освоить `git push` для отправки изменений: «[git push: как отправить код в репозиторий]({{< relref "git-push" >}})».

## По теме
- [Форк репозитория]({{< relref "fork-repozitoriya" >}})

- [Приватный репозиторий GitHub]({{< relref "github-private-repository" >}})
