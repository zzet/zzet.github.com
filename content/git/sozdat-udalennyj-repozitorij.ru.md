---
title: "Как создать удалённый репозиторий Git: пошаговое руководство"
description: "Пошаговое руководство по созданию удалённого Git репозитория. Bare репозиторий на SSH сервере, локальный удалённый репо."
date: 2026-03-05
lastmod: 2026-03-05
draft: false
slug: "sozdat-udalennyj-repozitorij"
keywords: ["создание удалённого репозитория git", "как создать удалённый репозиторий", "remote repository", "создать remote репозиторий", "git remote создать", "создать репозиторий на сервере"]
tags: ["git", "remote", "intermediate"]
categories: ["git"]
---

Большинство разработчиков используют GitHub или GitLab для хостинга репозиториев. Но иногда нужен собственный сервер: для приватных проектов с жёсткими требованиями безопасности, для внутренней инфраструктуры без доступа в интернет, или просто для понимания того, как устроены удалённые репозитории изнутри.

## Что такое bare репозиторий

Обычный репозиторий Git имеет рабочую директорию (файлы проекта) и скрытую папку `.git` с историей. Bare (голый) репозиторий — только содержимое папки `.git` без рабочих файлов.

Зачем это нужно: в репозиторий, куда делают push несколько разработчиков, нельзя иметь рабочую директорию, иначе push сломает текущий checkout. Bare репозиторий — стандартный способ создания удалённого хранилища.

По соглашению, bare репозитории называются с суффиксом `.git`: `my-project.git`.

## Способ 1: Bare репозиторий на сервере по SSH

```bash
# Подключиться к серверу
ssh user@server.example.com

# Перейти в директорию для репозиториев
cd /var/git
# или
cd /home/git

# Создать директорию для проекта
mkdir my-project.git
cd my-project.git

# Инициализировать bare репозиторий
git init --bare

# Вывод:
# Initialized empty Git repository in /var/git/my-project.git/

# Выйти с сервера
exit
```

Теперь подключите локальный репозиторий к созданному:

```bash
# На локальной машине
cd ~/my-project

# Добавить remote
git remote add origin ssh://user@server.example.com/var/git/my-project.git
# или в сокращённом SSH формате
git remote add origin user@server.example.com:/var/git/my-project.git

# Проверить
git remote -v

# Отправить первый коммит
git push -u origin main
```

## Способ 2: Локальный «удалённый» репозиторий

Для обучения или локальной командной работы можно создать bare репозиторий прямо на своём компьютере:

```bash
# Создать директорию для «сервера»
mkdir ~/git-server
cd ~/git-server

# Создать bare репозиторий
mkdir my-project.git
cd my-project.git
git init --bare

# Подключить существующий проект
cd ~/my-project
git remote add origin ~/git-server/my-project.git
git push -u origin main

# Клонировать (как будто с другой машины)
cd ~/tmp
git clone ~/git-server/my-project.git my-project-clone
```

## Способ 3: GitHub/GitLab через веб-интерфейс

Для большинства команд проще создать репозиторий через веб-интерфейс:

На GitHub: кнопка «New repository» → выбрать имя → Create. Затем в локальном проекте:

```bash
git remote add origin https://github.com/username/my-project.git
git push -u origin main
```

На GitLab: «New project» → «Create blank project» → настроить видимость. URL подключения будет показан на странице проекта.

## Клонирование из нового репозитория

После создания bare репозитория другие разработчики могут клонировать:

```bash
# Клонировать по SSH
git clone ssh://user@server.example.com/var/git/my-project.git

# Клонировать в директорию с другим именем
git clone ssh://user@server.example.com/var/git/my-project.git local-name

# Клонировать локальный bare репозиторий
git clone ~/git-server/my-project.git
```

## Настройка прав доступа

Для многопользовательского доступа к bare репозиторию на сервере:

```bash
# На сервере: создать пользователя git
sudo useradd -m git
sudo mkdir /home/git/.ssh
sudo chmod 700 /home/git/.ssh

# Добавить публичные ключи разработчиков
sudo nano /home/git/.ssh/authorized_keys
# Добавить каждый публичный ключ отдельной строкой

# Установить права на директорию репозитория
sudo chown -R git:git /var/git
sudo chmod -R 775 /var/git

# Теперь разработчики подключаются как:
git clone git@server.example.com:/var/git/my-project.git
```

## Создание репозитория на GitHub через веб-интерфейс

Шаг за шагом через GitHub UI:

```
1. Зайти на github.com → кнопка "+" в правом верхнем углу
2. Выбрать "New repository"
3. Заполнить форму:
   - Repository name: "my-project"
   - Description: "Проект для управления задачами"
   - Public или Private (выбрать)
   - Initialize with README? (опционально)
   - Add .gitignore? (выбрать язык)
   - Add license? (опционально)
4. Нажать "Create repository"
5. Скопировать HTTPS или SSH URL
```

После создания:

```bash
# Способ 1: Если репо создано пустым
cd ~/my-project
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/username/my-project.git
git push -u origin main

# Способ 2: Если репо создано с README
git clone https://github.com/username/my-project.git
cd my-project
# готово к работе
```

## Создание репозитория через GitHub CLI

Быстрый способ через `gh` команду:

```bash
# Установить GitHub CLI
# https://cli.github.com

# Логин в GitHub
gh auth login

# Создать новый репозиторий и клонировать
gh repo create my-project --source=. --remote=origin --push

# Создать только на GitHub без локального
gh repo create my-project --public

# Создать приватный репозиторий
gh repo create my-project --private

# Создать в определённой организации
gh repo create my-org/my-project --public
```

Параметры `gh repo create`:

```bash
gh repo create [<repository>] [flags]

Flags:
  -d, --description string       Description of repository
  -s, --source string            Repository to use as template
  --public                       Make repository public
  --private                      Make repository private
  --internal                     Make repository internal (enterprise only)
  --enable-issues bool           Enable issues (default true)
  --enable-projects bool         Enable projects (default true)
  --enable-wiki bool             Enable wiki (default true)
  --allow-auto-merge bool        Allow auto merge commits
  --allow-squash-merge bool      Allow squash merges (default true)
  --allow-rebase-merge bool      Allow rebase merges (default true)
  --delete-branch-on-merge bool  Delete branch on merge (default false)
```

## Создание репозитория на GitLab

Похоже на GitHub, но с отличиями:

```
GitLab UI:
1. Главная → "Create new project"
2. "Create blank project" (или use template)
3. Project name: "my-project"
4. Project slug: "my-project"
5. Visibility level: Private / Internal / Public
   (опции отличаются от GitHub)
6. Create project

Отличия от GitHub:
- Есть "Internal" уровень (видно членам организации)
- Более детальные настройки прав доступа
- Встроенный CI/CD (GitLab CI)
- Встроенный registry для Docker
```

Подключение локально:

```bash
git remote add origin https://gitlab.com/username/my-project.git
git branch -M main
git push -u origin main
```

## Создание репозитория на Bitbucket

Для тех кто использует Atlassian:

```bash
# На bitbucket.org:
# 1. Workspace → Create repository
# 2. Repository name, Access level (Private/Public)
# 3. Scm (Git/Mercurial) → Git
# 4. Create repository

# Локально:
git remote add origin https://bitbucket.org/username/my-project.git
git push -u origin main
```

## Репозиторий: Private vs Public

Рекомендации по выбору:

```
Выбирайте PRIVATE если:
✓ Коммерческий проект
✓ Чувствительные данные
✓ Приватный GitHub Codespaces
✓ Приватные DevOps скрипты
✓ Не готовы к открытому коду

Выбирайте PUBLIC если:
✓ Open source проект
✓ Портфолио для работодателей
✓ Образовательный проект
✓ Инструмент для других разработчиков
✓ Нет проблем если код скопировать
```

Изменить видимость позже:

```bash
# GitHub UI: Settings → Danger Zone → Change visibility
# GitLab UI: Settings → General → Visibility
```

## Инициализация README, .gitignore и License

При создании репозитория можно выбрать:

**README.md:**
```markdown
# My Project

Краткое описание проекта

## Установка

```bash
git clone ...
cd my-project
npm install
```

## Использование

...

## Лицензия

MIT
```

**.gitignore по языку:**
```
Выбрать при создании:
- Node
- Python
- Java
- Go
- и т.д.

Или добавить вручную:
git clone https://github.com/github/gitignore.git
cp gitignore/Node.gitignore .gitignore
git add .gitignore
git commit -m "Add .gitignore"
```

**Лицензия:**
```
Популярные лицензии:
- MIT (простая, требует указание авторства)
- Apache 2.0 (явно покрывает patent rights)
- GPL 3.0 (open source код остаётся open source)
- BSD (MIT с доп гарантиями)
- Proprietary (свой текст лицензии)

GitHub автоматически создаст файл LICENSE.md
```

## Инициализация существующего проекта

```bash
# Если у вас есть локальный проект без Git
cd ~/existing-project

# Инициализировать локальный репо
git init
git add .
git commit -m "Initial commit"

# Создать bare репозиторий на сервере
ssh user@server.example.com "mkdir -p /var/git/my-project.git && git init --bare /var/git/my-project.git"

# Добавить remote и отправить
git remote add origin user@server.example.com:/var/git/my-project.git
git push -u origin main
```

## Практические примеры

```bash
# Создать bare репозиторий одной командой
ssh user@server "git init --bare /var/git/project.git"

# Клонировать с SSH
git clone git@server.example.com:/var/git/project.git

# Просмотр содержимого bare репозитория
ls /var/git/project.git
# HEAD  branches  config  description  hooks  info  objects  packed-refs  refs

# Изменить URL после переноса репозитория
git remote set-url origin ssh://new-server/var/git/project.git

# Синхронизировать старый и новый сервер
git clone --mirror old-server:/var/git/project.git
cd project.git
git remote set-url --push origin new-server:/var/git/project.git
git push --mirror
```

## Трансфер владения репозиторием

Если нужно передать репозиторий другому пользователю:

**GitHub:**
```
1. Settings → Danger Zone → Transfer ownership
2. Выбрать нового владельца
3. Ввести имя репозитория для подтверждения
4. Нажать "I understand, transfer this repository"
```

**GitLab:**
```
1. Settings → General → Advanced
2. "Transfer project"
3. Выбрать группу/пользователя
4. Нажать "Transfer project"
```

После трансфера:

```bash
# Обновить remote URL в локальном репо
git remote set-url origin https://github.com/new-owner/my-project.git

# Проверить
git remote -v
```

## Клонирование сразу после создания

После создания репозитория на GitHub/GitLab:

```bash
# Вариант 1: Клонировать пустой репо
git clone https://github.com/username/my-project.git
cd my-project
# Добавить файлы
git add .
git commit -m "Initial commit"
git push origin main

# Вариант 2: Если репо уже содержит README
git clone https://github.com/username/my-project.git
cd my-project
# Уже готово к работе

# Вариант 3: Клонировать с другим именем локальной папки
git clone https://github.com/username/my-project.git my-local-folder
cd my-local-folder
```

## Настройки репозитория после создания

Рекомендуемые настройки:

**GitHub Settings → General:**
```
✓ Default branch: main
✓ Require status checks before merging: ON
✓ Dismiss approved reviews: ON
✓ Require code review before merging: ON
✓ Require number of approvals: 1
✓ Allow auto-merge: ON (если используете)
✓ Always suggest updating PR branches: ON
```

**GitHub Settings → Branches:**
```
1. Add rule
2. Branch name pattern: main
3. Require pull request reviews before merging
4. Require status checks to pass
5. Require branches to be up to date
6. Include administrators: ON
```

**GitHub Settings → Collaborators & teams:**
```
Добавить членов команды с нужными правами:
- Pull only: для внешних контрибьюторов
- Triage: для тех кто может labeling
- Write: для разработчиков
- Maintain: для lead разработчиков
- Admin: для владельцев
```

**GitHub Settings → Secret and variables:**
```
Добавить GitHub secrets для CI/CD:
- DEPLOY_KEY: SSH ключ для деплоя
- DOCKER_TOKEN: Docker Hub credentials
- API_TOKEN: Внешний API ключ
```

## Подключение существующего локального репо к новому удалённому

```bash
# У вас есть локальный репо без remote

# Способ 1: Добавить remote
cd ~/my-existing-project
git remote add origin https://github.com/username/my-project.git
git branch -M main
git push -u origin main

# Способ 2: Если у вас develop ветка
git branch -M develop main  # переименовать
git push -u origin main

# Способ 3: Если у вас несколько веток
git branch -a  # посмотреть все ветки
git push -u origin main
git push -u origin develop
git push --all origin
git push --tags origin
```

## Зеркалирование между хостингами

Если нужно синхронизировать репозиторий между GitHub и GitLab:

```bash
# Способ 1: Using mirror clone
git clone --mirror https://github.com/username/project.git
cd project.git
git push --mirror https://gitlab.com/username/project.git

# Способ 2: Синхронизировать локально
git clone https://github.com/username/project.git
cd project
git remote add gitlab https://gitlab.com/username/project.git
git push gitlab main
git push gitlab --all
git push gitlab --tags

# Способ 3: GitHub Actions для автосинхронизации
# .github/workflows/mirror.yml
name: Mirror to GitLab
on:
  push:
    branches: ['**']

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Push to GitLab
        run: git push --mirror https://oauth2:${{ secrets.GITLAB_TOKEN }}@gitlab.com/username/project.git
```

## Часто задаваемые вопросы

**Что такое bare репозиторий и почему он необходим?** Bare репозиторий содержит только данные Git (историю, ветки, теги) без рабочей директории. Это стандарт для удалённых репозиториев, так как в них нельзя иметь рабочие файлы — иначе push сломает текущее состояние.

**Можно ли использовать обычный репозиторий вместо bare?** Технически да, но push будет выдавать предупреждения, а текущая ветка не обновится. Для многопользовательского использования всегда используйте `--bare`.

**Как дать доступ к репозиторию другим пользователям?** Через SSH: добавить их публичные ключи в `~/.ssh/authorized_keys` пользователя сервера. Через HTTP: нужен Git HTTP backend (git-http-backend) или GitLab/Gitea.

**Какой протокол использовать: SSH или HTTP?** SSH удобнее для разработчиков (не нужно вводить пароль с настроенными ключами), надёжнее и безопаснее. HTTP лучше для публичного доступа на чтение.

**Как настроить SSH доступ к удалённому репозиторию?** Настройте SSH-ключи: `ssh-keygen`, добавьте публичный ключ на сервер в `authorized_keys`. Подробнее — в статье о [подключении по SSH]({{< relref "podklyuchenie-k-git-po-ssh" >}}).

## Заключение

Создать удалённый Git репозиторий несложно: `git init --bare` на сервере, `git remote add` в локальном проекте, `git push -u origin main`. Для большинства команд GitHub или GitLab удобнее и не требуют собственной инфраструктуры. Для управления remote репозиториями используйте [git remote add]({{< relref "git-remote-add" >}}) и [git remote -v]({{< relref "git-remote-v" >}}).
