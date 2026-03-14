---
title: "Собственный Git сервер: настройка self-hosted хостинга репозиториев"
description: "Как развернуть собственный Git сервер. SSH доступ, git daemon, Gitea, GitLab CE, Gitolite. Сравнение вариантов self-hosted Git."
date: 2026-03-03
lastmod: 2026-03-03
draft: false
slug: "sobstvennyj-git-server"
keywords: ["собственный git сервер", "self-hosted git", "gitea установка", "gitlab ce сервер"]
tags: ["git", "advanced"]
categories: ["git"]
---

Развернуть собственный Git сервер нужно, когда данные нельзя хранить в облачных сервисах, нужен полный контроль над репозиториями, или просто хочется избежать зависимости от GitHub/GitLab.com. Вариантов несколько — от минималистичного SSH до полноценных веб-платформ.

## Вариант 1: простой SSH сервер

Самый простой способ — Git репозитории через SSH. Не требует дополнительного ПО.

### Настройка сервера

```bash
# На сервере: создать пользователя git
sudo adduser git
sudo su - git

# Настроить SSH ключи
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Добавить публичный ключ разработчика
echo "ssh-ed25519 AAAA... developer@laptop" >> ~/.ssh/authorized_keys

# Создать репозиторий
mkdir -p ~/repos
cd ~/repos
git init --bare myapp.git
```

### Работа с репозиторием

```bash
# Разработчик: клонировать репозиторий
git clone git@server.example.com:repos/myapp.git
cd myapp

# Обычная работа
echo "Hello" > README.md
git add README.md
git commit -m "Initial commit"
git push origin main

# Клонировать у другого разработчика
git clone git@server.example.com:repos/myapp.git
```

### Ограничение доступа

```bash
# Запретить git пользователю интерактивный shell
sudo chsh -s $(which git-shell) git

# Теперь: git push/clone работают
# Но: ssh git@server.example.com — отказано
# git-shell разрешает только Git операции

# Разрешить только определённые команды в authorized_keys
cat ~/.ssh/authorized_keys
# command="git-shell -c \"$SSH_ORIGINAL_COMMAND\"",
# no-port-forwarding,no-X11-forwarding,no-agent-forwarding
# ssh-ed25519 AAAA... developer@laptop
```

## Вариант 2: Gitea (рекомендуется)

Gitea — лёгкий self-hosted Git сервис с веб-интерфейсом. Один бинарный файл, низкие требования к ресурсам.

```bash
# Установка через Docker (проще всего)
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -p 222:22 \
  -v /srv/gitea:/data \
  gitea/gitea:latest

# Или через docker-compose
cat > docker-compose.yml << 'EOF'
version: "3"
services:
  gitea:
    image: gitea/gitea:latest
    environment:
      - USER_UID=1000
      - USER_GID=1000
    ports:
      - "3000:3000"
      - "222:22"
    volumes:
      - ./gitea:/data
    restart: unless-stopped
EOF
docker-compose up -d

# Открыть http://server:3000 и пройти первоначальную настройку
```

Возможности Gitea: веб-интерфейс как GitHub, pull requests, issues, CI/CD (Gitea Actions), организации, разграничение прав, webhooks, API.

Минимальные требования: 512 МБ RAM, 1 ядро CPU, подходит для небольших команд.

## Вариант 3: GitLab Community Edition

GitLab CE — полноценная DevOps платформа с бесплатным self-hosted вариантом.

```bash
# Установка на Ubuntu 22.04
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://gitlab.example.com" apt install gitlab-ce

# Первый вход: пользователь root, пароль в файле
sudo cat /etc/gitlab/initial_root_password

# Конфигурация в /etc/gitlab/gitlab.rb
sudo gitlab-ctl reconfigure  # применить изменения

# Обновление
sudo apt update && sudo apt upgrade gitlab-ce
```

GitLab CE включает: репозитории, CI/CD pipelines, container registry, issue tracker, wiki, code review. Требует: минимум 4 ГБ RAM, 4 ядра CPU, 10 ГБ диска.

## Вариант 4: Forgejo

Forgejo — fork Gitea с открытым управлением. API совместим с Gitea:

```bash
docker run -d \
  --name forgejo \
  -p 3000:3000 \
  -p 222:22 \
  -v /srv/forgejo:/data \
  codeberg.org/forgejo/forgejo:latest
```

## Вариант 5: Gitolite (управление правами)

Gitolite — система управления доступом к Git репозиториям через SSH:

```bash
# Установка Gitolite (на сервере)
sudo apt install gitolite3

# Администрирование через специальный репозиторий
git clone git@server:gitolite-admin
cd gitolite-admin

# conf/gitolite.conf — конфигурация прав
cat conf/gitolite.conf
```

```
repo myapp
    RW+     =   alice bob   # полный доступ
    R       =   charlie     # только чтение
    -       =   dave        # запрет

repo @all
    R       =   admin       # admin читает все репозитории
```

```bash
# Добавить нового разработчика
cp /tmp/dave.pub keydir/dave.pub
git add keydir/dave.pub
git commit -m "Add dave's SSH key"
git push  # изменения применяются автоматически
```

## Сравнение вариантов

```
Вариант         RAM    Сложность   Возможности
SSH + bare      ~0     Низкая      Только репозитории
Gitea           512МБ  Низкая      Веб-интерфейс, PR, Issues, CI
Forgejo         512МБ  Низкая      Как Gitea, более открытый
GitLab CE       4ГБ    Высокая     Полная DevOps платформа
Gitolite        ~0     Средняя     Тонкое управление правами
```

## Резервное копирование

Независимо от выбранного варианта, регулярное резервное копирование обязательно:

```bash
# Простое зеркалирование bare репозитория
rsync -avz /srv/git/repos/ backup-server:/backup/git/

# Или через git clone --mirror
git clone --mirror git@server:repos/myapp.git /backup/myapp.git
# Обновление зеркала:
cd /backup/myapp.git && git remote update

# Для GitLab: встроенный бэкап
sudo gitlab-backup create
# Файл: /var/opt/gitlab/backups/XXXXXXXXX_gitlab_backup.tar

# Для Gitea: бэкап через утилиту
gitea dump -c /etc/gitea/app.ini
```

## Webhooks для CI/CD

```bash
# После push на сервер: автодеплой через post-receive хук
cat > /srv/git/repos/myapp.git/hooks/post-receive << 'EOF'
#!/bin/bash
while read oldrev newrev refname; do
  if [ "$refname" = "refs/heads/main" ]; then
    echo "Deploying main branch..."
    # Обновить рабочую копию
    git --work-tree=/var/www/myapp --git-dir=/srv/git/repos/myapp.git checkout -f main
    # Или вызвать скрипт деплоя
    /usr/local/bin/deploy-myapp.sh
  fi
done
EOF
chmod +x /srv/git/repos/myapp.git/hooks/post-receive
```

## Часто задаваемые вопросы

**Что выбрать для команды из 5 человек?** Gitea — оптимальный выбор. Лёгкий, с веб-интерфейсом, pull requests, и не требует мощного сервера. Запускается в Docker за 5 минут.

**Нужен ли SSL/TLS для собственного сервера?** Да. Без TLS пароли и токены передаются открытым текстом. Let's Encrypt предоставляет бесплатные сертификаты: установите nginx перед Gitea с certbot.

**Как мигрировать репозитории с GitHub?** Gitea и GitLab имеют встроенные инструменты импорта. Или вручную: `git clone --mirror github.com/user/repo.git && git push --mirror git@myserver:user/repo.git`.

**Можно ли использовать GitHub Actions с собственным сервером?** Нет. Но Gitea поддерживает Gitea Actions (совместимо с GitHub Actions синтаксисом). GitLab CE имеет встроенный CI/CD.

**Как разграничить права если нужна тонкая настройка?** GitLab CE предоставляет роли (Owner/Maintainer/Developer/Reporter/Guest) на уровне проекта и группы. Gitolite предоставляет настройку прав вплоть до отдельных веток через конфигурационный файл.

## Заключение

Для минималистичного решения — SSH с bare репозиторием. Для командной разработки с удобным интерфейсом — Gitea (легковесный) или GitLab CE (полная платформа). Выбор зависит от ресурсов сервера и нужных функций. В любом случае настройте резервное копирование и HTTPS. О [протоколах Git]({{< relref "protokoly-git" >}}) читайте в отдельной статье.
