---
title: "Настройка SSH для GitHub: пошаговое руководство"
description: "Как создать SSH-ключи и добавить их в GitHub для безопасной работы без пароля. Пошаговая инструкция."
date: 2026-02-16
lastmod: 2026-02-16
draft: false
slug: "nastrojka-ssh-github"
keywords: ["github подключение по ssh", "настройка git ssh", "ssh git clone", "ssh ключ github", "добавить ssh ключ github", "ssh-keygen github"]
tags: ["git", "github", "ssh", "setup"]
categories: ["git"]
aliases: []
---

SSH (Secure Shell) — это безопасный протокол для работы с удалёнными серверами. Вместо ввода пароля каждый раз при `git push` или `git pull` вы используете пару криптографических ключей. Настройка занимает около 10 минут, а работать станет значительно удобнее.

Прежде чем начать, убедитесь, что Git уже установлен и настроен. Если нет — прочитайте статью «[Установка и настройка Git]({{< relref "nastrojka-git" >}})».

## Что такое SSH и почему это лучше пароля

SSH работает на основе пары ключей:

**Приватный ключ** (private key) — хранится только на вашем компьютере. Никому его не показывайте и никуда не загружайте.

**Публичный ключ** (public key) — добавляется на сервер (GitHub, GitLab и т.д.). Он не секретный — его можно свободно передавать.

Когда вы подключаетесь к GitHub, сервер проверяет, есть ли у вас приватный ключ, соответствующий зарегистрированному публичному. Если есть — пускает без пароля.

Преимущества SSH перед HTTPS с паролем:
- Не нужно вводить пароль при каждом `git push`/`git pull`
- Ключи значительно сложнее угадать или подобрать, чем пароль
- Один ключ работает с GitHub, GitLab, Bitbucket и другими сервисами
- Это стандарт индустрии для серверной работы с Git

## Шаг 1: проверка наличия существующих ключей

Сначала проверьте, нет ли уже SSH ключей:

```bash
ls -la ~/.ssh
```

Если видите файлы `id_ed25519` и `id_ed25519.pub` (или `id_rsa` и `id_rsa.pub`) — ключи уже есть. Переходите к Шагу 3.

Если папка `.ssh` не существует или пустая — создайте новые ключи.

## Шаг 2: генерация нового SSH ключа

Создайте пару ключей с помощью `ssh-keygen`:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Алгоритм `ed25519` современный и рекомендуемый. Если ваша система очень старая и не поддерживает ed25519:
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

Параметры команды:
- `-t ed25519` — тип ключа (эллиптическая криптография, быстрый и безопасный)
- `-C "email"` — комментарий для идентификации ключа

### Диалог при создании ключа

```
Enter file in which to save the key (/Users/user/.ssh/id_ed25519):
```
Нажмите Enter, чтобы сохранить в стандартное место. Или укажите другой путь, если хотите отдельный ключ для GitHub.

```
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```
Passphrase (пароль для ключа) — необязательна. С ней ключ защищён дополнительно: даже если кто-то получит файл ключа, без пароля он бесполезен. Без passphrase ключ работает автоматически.

После создания появятся два файла:
- `~/.ssh/id_ed25519` — приватный ключ (не передавайте никому!)
- `~/.ssh/id_ed25519.pub` — публичный ключ (добавляется на GitHub)

## Шаг 3: добавление публичного ключа в GitHub

### 3.1 Скопируйте публичный ключ

```bash
# macOS — копирует в буфер обмена
cat ~/.ssh/id_ed25519.pub | pbcopy

# Linux (требует xclip)
cat ~/.ssh/id_ed25519.pub | xclip -selection clipboard

# Linux (требует xsel)
cat ~/.ssh/id_ed25519.pub | xsel --clipboard --input

# Windows (PowerShell)
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard

# Или просто выведите и скопируйте вручную
cat ~/.ssh/id_ed25519.pub
```

Вывод будет выглядеть примерно так:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your_email@example.com
```

Скопируйте всю строку целиком.

### 3.2 Добавьте ключ в GitHub

1. Откройте [GitHub.com](https://github.com) и войдите в аккаунт
2. Нажмите на аватар → **Settings**
3. В левом меню выберите **SSH and GPG keys**
4. Нажмите **New SSH key**
5. В поле **Title** укажите название: «My Laptop», «Work MacBook» и т.д.
6. В поле **Key** вставьте скопированный публичный ключ
7. Нажмите **Add SSH key**

## Шаг 4: добавление ключа в SSH-агент

Если вы задали passphrase, при каждом обращении к GitHub Git будет её запрашивать. SSH-агент позволяет ввести passphrase один раз за сеанс:

**macOS:**
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

**Linux:**
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

**Windows (Git Bash):**
```bash
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_ed25519
```

Для автоматического добавления ключа при входе в macOS создайте или отредактируйте файл `~/.ssh/config`:
```
Host github.com
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519
```

## Шаг 5: проверка подключения

Убедитесь, что SSH работает:

```bash
ssh -T git@github.com
```

Первый раз появится предупреждение:
```
The authenticity of host 'github.com (140.82.114.4)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
Are you sure you want to continue connecting (yes/no)?
```

Введите `yes`. Если всё настроено правильно:
```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

Это означает, что SSH работает корректно.

## Шаг 6: использование SSH при клонировании

Теперь при клонировании используйте SSH URL вместо HTTPS:

```bash
# SSH (рекомендуется)
git clone git@github.com:username/repository.git

# HTTPS (не рекомендуется после настройки SSH)
git clone https://github.com/username/repository.git
```

SSH URL начинается с `git@`, HTTPS — с `https://`. На GitHub при копировании ссылки нажмите **SSH** чтобы получить правильный адрес.

### Изменение URL в существующем репозитории

Если репозиторий уже клонирован через HTTPS:

```bash
# Проверьте текущий URL
git remote -v
# origin  https://github.com/username/repository.git (fetch)
# origin  https://github.com/username/repository.git (push)

# Измените на SSH
git remote set-url origin git@github.com:username/repository.git

# Проверьте результат
git remote -v
# origin  git@github.com:username/repository.git (fetch)
# origin  git@github.com:username/repository.git (push)
```

## Настройка для нескольких аккаунтов

Если у вас несколько аккаунтов (личный и рабочий), создайте отдельные ключи:

```bash
# Личный аккаунт
ssh-keygen -t ed25519 -C "personal@email.com" -f ~/.ssh/id_github_personal

# Рабочий аккаунт
ssh-keygen -t ed25519 -C "work@company.com" -f ~/.ssh/id_github_work
```

Затем настройте файл `~/.ssh/config`:

```
# Личный GitHub
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_github_personal

# Рабочий GitHub
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_github_work
```

Теперь при клонировании используйте псевдоним хоста:

```bash
# Личный репозиторий
git clone git@github-personal:username/personal-repo.git

# Рабочий репозиторий
git clone git@github-work:company/work-repo.git
```

## Решение частых проблем

### Ошибка «Permission denied (publickey)»

```bash
git push
# git@github.com: Permission denied (publickey).
```

Проверьте:
1. Добавлен ли публичный ключ в GitHub Settings → SSH keys
2. Правильный ли файл ключа: `ls ~/.ssh/`
3. Правильные ли права доступа: `chmod 600 ~/.ssh/id_ed25519`
4. Работает ли SSH-агент: `ssh-add -l`

### Ошибка «Could not read from remote repository»

```bash
# Проверьте SSH подключение
ssh -T git@github.com

# Проверьте URL репозитория
git remote -v
```

Убедитесь, что используете SSH URL (`git@github.com:`), а не HTTPS (`https://github.com/`).

### Неверный fingerprint

GitHub периодически обновляет SSH-fingerprint. Проверьте актуальные значения на [документации GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints).

## Полный пример настройки за одну сессию

```bash
# 1. Создание ключа
ssh-keygen -t ed25519 -C "your_email@example.com"
# Нажмите Enter для стандартного пути
# Введите passphrase (необязательно)

# 2. Просмотр публичного ключа
cat ~/.ssh/id_ed25519.pub
# Скопируйте вывод

# 3. Добавьте ключ в GitHub (вручную через веб-интерфейс)
# Settings → SSH and GPG keys → New SSH key → вставить → Add SSH key

# 4. Добавление ключа в ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 5. Проверка подключения
ssh -T git@github.com
# Hi username! You've successfully authenticated...

# 6. Клонирование по SSH
git clone git@github.com:username/my-repo.git
cd my-repo

# 7. Обычная работа
git add .
git commit -m "changes"
git push
```

## Часто задаваемые вопросы

**Можно ли один SSH-ключ использовать для GitHub и GitLab?** Да. Добавьте один и тот же публичный ключ в оба сервиса. Файл `~/.ssh/config` можно не настраивать.

**Что если я потеряю приватный ключ?** Создайте новую пару ключей. Удалите старый публичный ключ из GitHub Settings → SSH keys и добавьте новый.

**Нужен ли пароль на SSH ключ?** Это необязательно, но рекомендуется для повышения безопасности. С SSH-агентом пароль вводится один раз при входе в систему.

**Почему GitHub не принимает мой ключ?** Убедитесь, что вы скопировали полное содержимое `.pub`-файла, включая начало `ssh-ed25519 AAAA...` и комментарий в конце.

**Как удалить старый ключ?** Откройте GitHub Settings → SSH and GPG keys → нажмите Delete рядом с нужным ключом.

## Заключение

SSH — стандартный способ работы с GitHub. Его настройка занимает несколько минут, а удобство использования несравнимо с постоянным вводом пароля. После настройки вы сможете спокойно работать с `git push` и `git pull` без лишних шагов.

Помните: приватный ключ — это ваш пароль. Никогда не загружайте файл `id_ed25519` (без `.pub`) на GitHub, в чат или куда-либо ещё.

## По теме

- [Приватный репозиторий GitHub]({{< relref "github-private-repository" >}})
