---
title: "Подключение к Git-серверу по SSH: полное руководство"
description: "Инструкция по подключению к Git-серверам (GitHub, GitLab, Bitbucket) по SSH, конфигурация, отладка проблем подключения."
date: 2026-02-25
lastmod: 2026-02-25
draft: false
slug: "podklyuchenie-k-git-po-ssh"
keywords: ["подключение по ssh git", "подключение к git по ssh", "git ssh ключ", "github ssh подключение"]
tags: ["git", "ssh", "github", "gitlab"]
categories: ["git"]
---

SSH позволяет безопасно подключаться к Git-серверам без ввода пароля при каждой операции. Это может быть GitHub, GitLab, Bitbucket или собственный сервер компании. Статья покажет как настроить SSH подключение для разных сервисов, использовать несколько ключей и решать типичные проблемы.

## Как работает SSH подключение к Git

SSH использует асимметричное шифрование с парой ключей: приватный ключ хранится на вашем компьютере, публичный — добавляется на сервер.

Процесс аутентификации:
1. Вы отправляете запрос на сервер (push или clone)
2. Сервер присылает «вызов» — случайные данные
3. Вы подписываете их своим приватным ключом
4. Сервер проверяет подпись с помощью вашего публичного ключа
5. Если совпадает — доступ разрешён, пароль не нужен

Это безопаснее и удобнее HTTPS с паролем: приватный ключ никогда не покидает ваш компьютер.

## SSH для разных Git-сервисов

**GitHub:**

```bash
# Создать ключ
ssh-keygen -t ed25519 -C "your_email@example.com"

# Скопировать публичный ключ
cat ~/.ssh/id_ed25519.pub

# Добавить в GitHub: Settings → SSH and GPG keys → New SSH key

# Клонировать по SSH
git clone git@github.com:username/repo.git
```

**GitLab:**

```bash
# Ключ создаётся так же
ssh-keygen -t ed25519 -C "your_email@example.com"

# Добавить в GitLab: Settings → SSH Keys

# Клонировать по SSH
git clone git@gitlab.com:username/repo.git
```

**Bitbucket:**

```bash
# Ключ создаётся аналогично
# Добавить в: Personal settings → SSH keys

git clone git@bitbucket.org:username/repo.git
```

**Собственный Git-сервер:**

```bash
# Добавить публичный ключ на сервер
cat ~/.ssh/id_ed25519.pub | ssh user@server.com "cat >> ~/.ssh/authorized_keys"

git clone ssh://git@server.com/path/to/repo.git
```

## Проверка SSH подключения

После настройки обязательно проверьте подключение:

```bash
# Проверка GitHub
ssh -T git@github.com
# Hi username! You've successfully authenticated, but GitHub does not provide shell access.

# Проверка GitLab
ssh -T git@gitlab.com
# Welcome to GitLab, @username!

# Проверка собственного сервера
ssh -T git@your-server.com
```

Если получаете `Permission denied (publickey)` — ключ не добавлен на сервер или используется неправильный ключ.

## Несколько SSH ключей для разных сервисов

Если у вас несколько аккаунтов (например, личный GitHub и рабочий GitLab), нужно настроить файл `~/.ssh/config`:

```bash
# Создать конфигурационный файл
nano ~/.ssh/config
```

```
# Личный GitHub
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_github_personal

# Рабочий GitLab
Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_gitlab_work

# Рабочий собственный сервер
Host company-git
    HostName git.company.com
    User git
    IdentityFile ~/.ssh/id_company
    Port 2222
```

Теперь SSH автоматически выбирает нужный ключ:

```bash
git clone git@github.com:personal/repo.git      # использует id_github_personal
git clone git@gitlab.com:work/project.git       # использует id_gitlab_work
git clone git@company-git:internal/code.git     # использует id_company
```

Для двух аккаунтов на одном сервере (например, два GitHub аккаунта):

```
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_personal

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_work
```

Используете разные алиасы:

```bash
git clone git@github-personal:personal-user/repo.git
git clone git@github-work:work-org/repo.git
```

## Изменение URL существующего репозитория на SSH

Если репозиторий уже клонирован через HTTPS:

```bash
# Посмотреть текущий URL
git remote -v
# origin  https://github.com/username/repository.git (fetch)
# origin  https://github.com/username/repository.git (push)

# Изменить URL на SSH
git remote set-url origin git@github.com:username/repository.git

# Проверить результат
git remote -v
# origin  git@github.com:username/repository.git (fetch)
# origin  git@github.com:username/repository.git (push)
```

## SSH-агент и пароли на ключи

Если SSH ключ защищён паролем (passphrase), его запрашивают при каждом подключении. SSH-агент запомнит пароль на время сеанса:

```bash
# Linux/macOS
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
# Введите пароль один раз — агент запомнит

# Windows (Git Bash)
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_ed25519

# Проверить загруженные ключи
ssh-add -l
```

На macOS можно добавить ключ в Keychain для автоматической загрузки:

```bash
# macOS — добавить в Keychain
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

## Права доступа на SSH ключи

SSH требователен к правам доступа. Если права неправильные, SSH отказывает в работе:

```bash
# Приватный ключ — только для владельца
chmod 600 ~/.ssh/id_ed25519

# Публичный ключ — можно читать всем
chmod 644 ~/.ssh/id_ed25519.pub

# Папка .ssh
chmod 700 ~/.ssh
```

## Отладка SSH подключения

Если что-то не работает:

```bash
# Стандартная проверка
ssh -T git@github.com

# Подробный вывод для отладки
ssh -v git@github.com

# Очень подробный вывод
ssh -vvv git@github.com
```

Подробный вывод покажет: какой ключ пробуется, какие методы аутентификации используются, где происходит ошибка.

## Подключение через нестандартный порт

Некоторые корпоративные серверы используют нестандартный порт:

```bash
# В ~/.ssh/config
Host git.company.com
    HostName git.company.com
    User git
    Port 2222
    IdentityFile ~/.ssh/id_company

# Или указать порт прямо в URL
git clone ssh://git@git.company.com:2222/repo.git
```

## Полный сценарий настройки подключения к GitHub

```bash
# 1. Проверить существующие ключи
ls -la ~/.ssh

# 2. Создать новый ключ
ssh-keygen -t ed25519 -C "your_email@example.com"
# Enter file: нажать Enter (использовать ~/ssh/id_ed25519)
# Passphrase: введите или оставьте пустой

# 3. Скопировать публичный ключ
cat ~/.ssh/id_ed25519.pub
# ssh-ed25519 AAAA... your_email@example.com

# 4. Добавить в GitHub
# Settings → SSH and GPG keys → New SSH key → вставить публичный ключ

# 5. Проверить подключение
ssh -T git@github.com
# Hi username! You've successfully authenticated

# 6. Клонировать репозиторий по SSH
git clone git@github.com:username/my-project.git

# 7. Работать как обычно
cd my-project
git add .
git commit -m "First commit"
git push
```

## Часто задаваемые вопросы

**SSH работает, но пишет "Permission denied (publickey)"** — проверьте: публичный ключ добавлен на сервер, права на приватный ключ `chmod 600`, SSH-агент запущен и ключ загружен (`ssh-add -l`).

**Как использовать SSH через корпоративный прокси?** Настройте `ProxyCommand` в `~/.ssh/config`: `ProxyCommand nc -x proxy.company.com:1080 %h %p`.

**Почему при первом подключении появляется вопрос о fingerprint?** SSH проверяет, что вы подключаетесь к правильному серверу (не к Man-in-the-Middle). Сравните fingerprint с официальным — для GitHub он опубликован в документации.

**Могу ли я использовать один ключ для нескольких сервисов?** Да, один публичный ключ можно добавить на GitHub и GitLab одновременно. Это удобно, но если ключ скомпрометирован — нужно менять везде.

**Что если я забыл passphrase ключа?** Восстановить его невозможно. Создайте новый ключ, добавьте его на все сервисы, удалите старый.

## Заключение

SSH подключение к Git-серверам надёжно и удобно. После первоначальной настройки вы больше не вводите пароль при каждом `git push` или `git pull`. Правильная конфигурация нескольких ключей позволяет работать с личными и рабочими проектами без путаницы.

Для более подробной инструкции по настройке SSH именно для GitHub читайте [руководство по настройке SSH для GitHub]({{< relref "nastrojka-ssh-github" >}}). Если вам нужно сравнение HTTPS и SSH протоколов — смотрите отдельную статью об этом.
