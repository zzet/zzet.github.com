---
title: "git init --bare: создание голого репозитория для совместной работы"
description: "Что такое bare репозиторий в Git. Создание через git init --bare, отличие от обычного репозитория, использование как центральный сервер для git push."
date: 2026-01-16
lastmod: 2026-01-16
draft: false
slug: "git-init-bare"
keywords: ["git init bare", "bare репозиторий git", "голый репозиторий git", "git init --bare что делает", "bare repository git", "создание сервера git bare"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Bare репозиторий (голый репозиторий) — Git репозиторий без рабочей директории. Он содержит только содержимое папки `.git`, но без файлов проекта. Bare репозитории используются как центральные серверы: в них можно делать `git push`, но нельзя работать напрямую.

## Что такое bare репозиторий

Обычный репозиторий состоит из двух частей:
- Рабочая директория (файлы проекта, которые вы редактируете)
- Git хранилище (папка `.git` со всей историей)

Bare репозиторий — только Git хранилище, без рабочей директории:

```
Обычный репозиторий:
my-project/
├── .git/         ← Git хранилище
├── src/
├── README.md
└── package.json

Bare репозиторий:
my-project.git/   ← только Git хранилище (без .git папки)
├── HEAD
├── config
├── objects/
└── refs/
```

## Создание bare репозитория

```bash
# Создать новый bare репозиторий
git init --bare my-project.git
# Initialized empty Git repository in /home/user/my-project.git/

# Содержимое bare репозитория
ls my-project.git/
# HEAD  config  description  hooks  info  objects  refs

# Конвертировать существующий репозиторий в bare
git clone --bare /path/to/my-project my-project.git
# Это создаёт bare копию существующего репозитория

# Создать bare с правами для группы (для сервера)
git init --bare --shared=group my-project.git
```

## Push в bare репозиторий

Главная особенность bare репозитория — в него можно делать `git push`:

```bash
# Настроить remote
git remote add origin /home/git/repos/my-project.git

# Push работает
git push origin main
# Counting objects: 3, done.
# Writing objects: 100% (3/3), 221 bytes | 221.00 KiB/s, done.
# To /home/git/repos/my-project.git
#  * [new branch]      main -> main

# Pull тоже работает
git pull origin main

# В обычный репозиторий с рабочей директорией push не работает
git push         # может привести к ошибке или непредсказуемому поведению
```

Почему нельзя делать push в обычный репозиторий? Потому что Git не знает что делать с рабочей директорией — у неё могут быть несохранённые изменения, и перезапись HEAD была бы опасна.

## Использование как локальный сервер

```bash
# Создать центральный репозиторий
mkdir -p /srv/git
cd /srv/git
git init --bare myapp.git

# Разработчик 1: клонировать
git clone /srv/git/myapp.git ~/myapp
cd ~/myapp
echo "Hello" > README.md
git add README.md
git commit -m "Initial commit"
git push origin main

# Разработчик 2: клонировать и получить изменения
git clone /srv/git/myapp.git ~/myapp2
cd ~/myapp2
git log --oneline
# abc1234 Initial commit  ← видит коммит от разработчика 1
```

## Использование по SSH

```bash
# На сервере (user: git, host: server.example.com)
ssh git@server.example.com
mkdir -p /home/git/repos
cd /home/git/repos
git init --bare myapp.git
exit

# Локально
git remote add origin git@server.example.com:/home/git/repos/myapp.git
git push origin main

# Клонировать
git clone git@server.example.com:/home/git/repos/myapp.git
```

## Конфигурация bare репозитория

Файл `config` в bare репозитории:

```bash
cat my-project.git/config
```

```ini
[core]
    repositoryformatversion = 0
    filemode = true
    bare = true              ← ключевой флаг
[receive]
    denyCurrentBranch = ignore
```

```bash
# Разрешить push непосредственно в проверяемую ветку
# (для обычных репозиториев, не bare)
git config receive.denyCurrentBranch updateInstead
# Тогда push обновит и рабочую директорию
```

## bare vs mirror

```bash
# --bare: голый репозиторий (для совместной работы)
git clone --bare https://github.com/user/project.git

# --mirror: зеркало (полная копия всех refs для резервного копирования)
git clone --mirror https://github.com/user/project.git
# Включает все ссылки: ветки, теги, refs/pull/, refs/notes/ и т.д.

# Обновить зеркало
cd project.git
git remote update  # или git fetch --all
```

## Проверка что репо является bare

Как узнать содержит ли репо рабочую директорию:

```bash
# Прямая проверка конфига
git rev-parse --is-bare-repository
# true  ← это bare репо
# false ← это обычный репо с рабочей директорией

# Также можно проверить файл config
cat my-project.git/config | grep bare
# bare = true

# Или посмотреть структуру:
ls my-project.git/
# Если есть только: HEAD, config, objects, refs, hooks, info
# и НЕТУ других файлов — это bare репо

ls my-project/
# Если видите: .git/, src/, README.md, package.json
# Это обычный репо
```

## git clone --bare vs git init --bare

Различие между двумя способами создания bare репо:

```bash
# git init --bare — создать пустой bare репо
git init --bare my-project.git
# Создаёт: HEAD, config, objects/, refs/, hooks/, info/
# Ничего больше

# git clone --bare — создать bare копию существующего репо
git clone --bare /path/to/my-project /tmp/my-project.git
# Копирует: всю историю, все ветки, все теги
# Но без рабочей директории

# Результат разный:
# init --bare:    пустой, готов для первого push
# clone --bare:   полная копия с историей

# Обновить bare копию (зеркало)
cd /tmp/my-project.git
git fetch origin
# или
git remote update
```

## Использование bare репо как backup

Bare репозиторий удобен для резервных копий:

```bash
# Способ 1: Прямая копия через clone --bare
git clone --bare /path/to/project /backup/project.git

# Способ 2: Периодическая синхронизация (cron job)
#!/bin/bash
# backup-git.sh
cd /backup/project.git
git fetch origin
git fetch --tags origin

# Добавить в crontab (каждый час)
0 * * * * /scripts/backup-git.sh

# Способ 3: Git как хранилище для CI/CD артефактов
# Использовать bare репо на сервере для хранения:
# - Release artifacts
# - Build outputs
git clone --bare . /ci-artifacts/build-$(date +%Y%m%d).git

# Способ 4: Mirror для full backup (все refs)
git clone --mirror https://github.com/user/project.git
cd project.git
git remote set-url origin https://github.com/user/project.git
git fetch --all  # обновить всё
```

## Post-receive hook для автоматического деплоя

Автоматизация деплоя при push:

```bash
# Создать hook
cat > /srv/git/myapp.git/hooks/post-receive << 'EOF'
#!/bin/bash

# Получить информацию о push
while read oldrev newrev refname; do
    echo "Received push to $refname ($oldrev -> $newrev)"

    # Если push в main ветку
    if [ "$refname" = "refs/heads/main" ]; then
        echo "Deploying main branch..."

        # Checkout в рабочую директорию
        git --work-tree=/var/www/myapp \
            --git-dir=/srv/git/myapp.git \
            checkout -f main

        # Запустить build скрипт
        cd /var/www/myapp
        ./scripts/deploy.sh

        # Отправить уведомление
        echo "Deploy complete!" | \
            mail -s "Deploy successful" admin@example.com
    fi
done
EOF

chmod +x /srv/git/myapp.git/hooks/post-receive
```

Параметры в post-receive hook:

```bash
# Получить информацию о пользователе
GIT_COMMITTER_NAME=$GL_USERNAME  # GitLab
REMOTE_USER=$USER                # SSH

# Получить список файлов в push
git diff-tree --no-commit-id -r $newrev

# Получить автора последнего коммита
git log -1 --format=%an $newrev

# Проверить какие файлы изменились
git diff --name-only $oldrev $newrev

# Отправить email со ссылкой на deploy
echo "Deployed: https://example.com/releases/$newrev"
```

## Bare репо для team collaboration

Настройка для совместной работы нескольких разработчиков:

```bash
# На сервере: создать пользователя git
sudo useradd -m git
sudo mkdir -p /home/git/.ssh

# Настроить права доступа
sudo chmod 700 /home/git/.ssh
sudo chown -R git:git /home/git/.ssh

# Добавить публичные ключи разработчиков
# Каждый разработчик генерирует ключ:
ssh-keygen -t ed25519 -f ~/.ssh/git_key -N ""

# Сервер: добавить ключи в authorized_keys
cat developer1-key.pub >> /home/git/.ssh/authorized_keys
cat developer2-key.pub >> /home/git/.ssh/authorized_keys

# Настроить права на директорию репо
sudo mkdir -p /srv/git
sudo chown git:git /srv/git
sudo chmod 770 /srv/git  # git пользователь и группа

# Создать bare репо
cd /srv/git
sudo -u git git init --bare --shared=group myapp.git

# Разработчики подключаются
git clone git@server:/srv/git/myapp.git
cd myapp

# Разработчик 1
git switch -c feature/login
git commit -am "Add login form"
git push origin feature/login

# Разработчик 2
git fetch origin
git switch -b feature/login origin/feature/login
# Или работать в своей ветке
git switch -c feature/auth
```

## Хуки в bare репозитории

Bare репозитории поддерживают серверные хуки:

```bash
# post-receive хук — срабатывает после push
cat > my-project.git/hooks/post-receive << 'EOF'
#!/bin/bash
echo "Received push, deploying..."
git --work-tree=/var/www/myapp --git-dir=/srv/git/myapp.git checkout -f main
echo "Deploy complete"
EOF
chmod +x my-project.git/hooks/post-receive

# Теперь при каждом push на сервер
# автоматически обновляется /var/www/myapp
```

## Работа с bare репозиторием напрямую

```bash
# Просмотр истории в bare репозитории
git --git-dir=/srv/git/myapp.git log --oneline

# Список веток
git --git-dir=/srv/git/myapp.git branch -a

# Просмотр файла из bare репозитория
git --git-dir=/srv/git/myapp.git show HEAD:README.md

# Создать ветку в bare репозитории
git --git-dir=/srv/git/myapp.git branch feature/new-ui main
```

## Различие между --bare и --mirror

Оба варианта создают репо без рабочей директории, но отличаются:

```bash
# git clone --bare
# Копирует: refs/heads/* (ветки)
#          refs/tags/* (теги)
# Игнорирует: refs/pull/* (GitHub pull requests)
#             refs/notes/* (git notes)
git clone --bare https://github.com/user/project.git

# git clone --mirror
# Копирует: ВСЕ refs
#          refs/heads/* (ветки)
#          refs/tags/* (теги)
#          refs/pull/* (pull requests как refs)
#          refs/notes/* (notes)
# Идеально для полного зеркала GitHub
git clone --mirror https://github.com/user/project.git

# Обновить bare репо
cd project.git
git fetch  # только новые ветки/теги

# Обновить mirror репо
cd project.git
git remote update  # всё, включая pull requests
```

**Выбор:**
- `--bare` для работающего репозитория
- `--mirror` для резервной копии или зеркала

## Конвертация обычного репо в bare

Если нужно преобразовать существующий репозиторий:

```bash
# Способ 1: Через clone --bare
cd /path/to/my-project
git clone --bare . ../my-project.git

# Способ 2: Вручную скопировать .git и изменить config
# Скопировать содержимое .git
cp -r /path/to/my-project/.git ../my-project.git

# Отредактировать config
cat ../my-project.git/config
# Изменить bare = true

# Или через команду
git config -f ../my-project.git/config core.bare true

# Способ 3: Из обычного репо сделать bare (в сам себе — опасно)
# НЕ ДЕЛАЙТЕ так! Используйте способы 1 или 2
# git --git-dir=.git config core.bare true
```

## Использование bare репо в CI/CD

Bare репозиторий часто используется в CI/CD пайплайнах:

```bash
# Gitlab Runner, GitHub Actions, Jenkins могут
# клонировать из bare репо без лишних затрат

# Пример: использовать как artifact store в CI
# .gitlab-ci.yml
stages:
  - build
  - push_cache

build:
  script:
    - ./build.sh
  artifacts:
    paths:
      - build/

push_cache:
  script:
    # Сохранить результат в bare репо (как cache)
    - git init --bare cache.git
    - git --git-dir=cache.git config receive.denyCurrentBranch ignore
    # Push build артефактов
    - tar czf build.tar.gz build/
    - git add build.tar.gz
    - git commit -m "Build cache"
    - git push cache.git main
```

## Производительность: bare vs обычный репо

Bare репозитории обычно быстрее для push операций:

```
Операция              Обычный репо    Bare репо
──────────────────────────────────────────────
git clone             медленнее        быстрее
git fetch             медленнее        быстрее
git push              медленнее        быстрее (нет RW проверок)
git pull              медленнее        недоступно
Локальная работа      ✓ быстро         ✗ невозможно
Размер на диске       +.git/ + files   только .git/
──────────────────────────────────────────────
```

Рекомендация:
- Разработка: обычный репо
- Сервер/backup: bare репо
- CI/CD artifact: bare репо

## Часто задаваемые вопросы

**Зачем соглашение о расширении .git для bare репозиториев?** Это просто соглашение (как GitHub и GitLab хранят репозитории), а не требование. `git init --bare project` работает так же хорошо.

**Можно ли клонировать из bare репозитория?** Да, `git clone /path/to/repo.git` работает как с обычными, так и с bare репозиториями.

**Почему нельзя редактировать файлы в bare репозитории?** Нет рабочей директории — негде делать изменения. Bare репозиторий существует только как точка обмена данными между разработчиками.

**Как сконвертировать обычный репозиторий в bare?** Скопируйте содержимое `.git/` в новую директорию и установите `core.bare = true` в config: `git clone --bare . ../project.git`. Исходный репозиторий при этом не изменяется.

**Чем отличается bare от зеркала GitHub?** `git clone --bare` копирует ветки и теги. `git clone --mirror` копирует абсолютно все ссылки, включая служебные (pull requests как refs/pull/*). Mirror используется для полного резервного копирования.

## Заключение

Bare репозиторий — Git хранилище без рабочей директории, предназначенное для центрального сервера. Создаётся через `git init --bare`. Позволяет делать `git push` от нескольких разработчиков. Используется при развёртывании [собственного Git сервера]({{< relref "sobstvennyj-git-server" >}}) и в хостинговых платформах (GitHub, GitLab хранят репозитории именно как bare).
