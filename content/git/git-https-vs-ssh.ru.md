---
title: "Git HTTPS vs SSH: как выбрать протокол для клонирования репозиториев"
description: "Сравнение HTTPS и SSH для Git: преимущества, недостатки, когда использовать каждый протокол. Как переключиться между ними."
date: 2026-01-15
lastmod: 2026-01-15
draft: false
slug: "git-https-vs-ssh"
keywords: ["git https", "ssh git clone", "git протокол", "https vs ssh git", "git port 443"]
tags: ["git", "ssh", "github"]
categories: ["git"]
---

При клонировании Git-репозитория GitHub предлагает два варианта: HTTPS и SSH. Оба работают одинаково с точки зрения функциональности, но имеют разные подходы к аутентификации. Эта статья поможет выбрать наиболее подходящий вариант и переключиться между ними при необходимости.

## Что такое HTTPS в контексте Git

HTTPS (HyperText Transfer Protocol Secure) — это тот же протокол, что используется в браузерах. При использовании для Git аутентификация происходит через имя пользователя и пароль (или персональный токен доступа).

```bash
# URL репозитория по HTTPS
https://github.com/username/repository.git

# Клонирование
git clone https://github.com/username/repository.git
```

GitHub и GitLab больше не принимают пароли для HTTPS — вместо них требуется Personal Access Token (PAT), который создаётся в настройках аккаунта.

## Что такое SSH в контексте Git

SSH (Secure Shell) — криптографический протокол для безопасного удалённого доступа. Аутентификация происходит через пару ключей: приватный ключ на вашем компьютере, публичный — на сервере.

```bash
# URL репозитория по SSH
git@github.com:username/repository.git

# Клонирование
git clone git@github.com:username/repository.git
```

После однократной настройки SSH ключей пароль при push/pull не требуется.

## Сравнение HTTPS и SSH

| Критерий | HTTPS | SSH |
|----------|-------|-----|
| Сложность настройки | Простая | Требует создания ключей |
| Аутентификация | Токен/пароль | SSH ключи |
| Пароль при каждой операции | Да (без кэша) | Нет |
| Безопасность | Хорошая | Отличная |
| Работает везде | Да | Обычно да |
| Скорость | Немного медленнее | Немного быстрее |
| Через корпоративный прокси | Лучше | Может быть заблокирован |

## HTTPS: плюсы и минусы

**Плюсы HTTPS:**

Не требует дополнительной настройки. Работает везде, где есть интернет и порт 443. Более знаком новичкам. Лучше проходит через корпоративные прокси. Легко использовать при работе на чужом компьютере.

**Минусы HTTPS:**

Нужно хранить персональный токен (PAT) — он длинный и неудобный. Без кэширования учётных данных токен запрашивается при каждой операции. Токены нужно периодически обновлять (при истечении срока действия).

```bash
# Сохранить учётные данные в кэш на 15 минут
git config --global credential.helper cache

# Сохранить учётные данные навсегда (в файл)
git config --global credential.helper store

# На macOS — хранить в Keychain
git config --global credential.helper osxkeychain
```

## SSH: плюсы и минусы

**Плюсы SSH:**

После настройки — полная автоматизация, никаких паролей. Ключи не истекают (если не удалить с сервера). Более безопасно: приватный ключ не передаётся по сети. Стандарт в профессиональной разработке и CI/CD.

**Минусы SSH:**

Требует первоначальной настройки (создание ключей и добавление на сервер). Нужно настраивать на каждом новом компьютере. Иногда блокируется корпоративными файрволами на порту 22.

## Когда использовать HTTPS

HTTPS подходит, если вы только начинаете с Git и не хотите заниматься настройкой ключей, работаете на общем или чужом компьютере, сеть блокирует SSH-соединения (порт 22), нужно быстро клонировать без локальной конфигурации.

## Когда использовать SSH

SSH рекомендуется, если вы работаете на личном компьютере, делаете много операций в день (push, pull, fetch), работаете в команде или профессиональной среде, настраиваете CI/CD пайплайны.

## Как переключиться между HTTPS и SSH

Если репозиторий уже клонирован, можно изменить протокол без повторного клонирования:

```bash
# Посмотреть текущий URL
git remote -v
# origin  https://github.com/username/repo.git (fetch)
# origin  https://github.com/username/repo.git (push)

# Переключить на SSH
git remote set-url origin git@github.com:username/repo.git

# Проверить результат
git remote -v
# origin  git@github.com:username/repo.git (fetch)
# origin  git@github.com:username/repo.git (push)

# Переключить обратно на HTTPS
git remote set-url origin https://github.com/username/repo.git
```

## Практические примеры

```bash
# Клонирование по HTTPS (простой способ для новичков)
git clone https://github.com/username/repo.git

# Клонирование по SSH (требует настройки SSH ключей)
git clone git@github.com:username/repo.git

# Переключение с HTTPS на SSH для существующего репозитория
cd repo
git remote set-url origin git@github.com:username/repo.git

# Проверка, какой протокол используется
git remote -v

# Дальнейшая работа одинакова независимо от протокола
git add .
git commit -m "changes"
git push
```

## Настройка Personal Access Token для HTTPS

GitHub больше не принимает обычные пароли. Вместо этого нужен Personal Access Token:

```bash
# На GitHub: Settings → Developer settings → Personal access tokens → Tokens (classic)
# Кликнуть "Generate new token" → "Generate new token (classic)"
# Выбрать scope: repo (для приватных репо) или public_repo (для публичных)
# Скопировать сгенерированный токен

# Использовать токен при клонировании
git clone https://USERNAME:TOKEN@github.com/username/repository.git

# Или при push (после первого клонирования по HTTPS)
git push
# Ввести username и вместо пароля вставить токен

# Сохранить токен локально (кэшировать на 15 минут)
git config --global credential.helper cache

# Или сохранить на диск (внимание: файл содержит токен!)
git config --global credential.helper store
# Потом Git запросит токен один раз и сохранит в ~/.git-credentials

# На macOS использовать Keychain
git config --global credential.helper osxkeychain
```

**Безопасность токенов:**
- Никогда не коммитьте токены в репозиторий
- Добавьте `.env` с токенами в `.gitignore`
- Регулярно ротируйте токены
- Используйте разные токены для разных целей (CI/CD, локальная разработка)

## Настройка SSH ключей

SSH требует начальной настройки, но после этого автоматизирует всё:

```bash
# Проверить есть ли уже ключи
ls -la ~/.ssh
# Если файлы id_ed25519 и id_ed25519.pub — ключи уже есть

# Создать новую пару ключей (если нет)
ssh-keygen -t ed25519 -C "your_email@example.com"
# или если система старая
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Вас спросят где сохранить — просто нажмите Enter для ~.ssh/id_ed25519
# Введите passphrase (пароль для ключа) или оставьте пусто

# Добавить ключ в SSH агент (на macOS/Linux)
ssh-add ~/.ssh/id_ed25519

# На Windows (PowerShell with admin)
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0

# Скопировать публичный ключ
cat ~/.ssh/id_ed25519.pub
# или
pbcopy < ~/.ssh/id_ed25519.pub  # macOS
xclip -selection clipboard < ~/.ssh/id_ed25519.pub  # Linux

# На GitHub: Settings → SSH and GPG keys → New SSH key
# Вставить содержимое id_ed25519.pub

# Проверить соединение
ssh -T git@github.com
# Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

## Хранение паролей и токенов безопасно

### Credential helpers

```bash
# Кэширование на время сессии
git config --global credential.helper cache

# Кэширование на час
git config --global credential.helper 'cache --timeout=3600'

# Хранение в файле (менее безопасно)
git config --global credential.helper store

# На macOS (самый безопасный вариант)
git config --global credential.helper osxkeychain

# На Windows
git config --global credential.helper wincred

# На Linux (если установлен pass)
git config --global credential.helper pass

# Проверить текущий helper
git config --global --get credential.helper

# Удалить сохранённые учётные данные
git credential-cache exit
# или
git credential approve  # введите неправильный пароль
```

## Использование SSH конфига для удобства

SSH можно настроить глобально в `~/.ssh/config`:

```bash
# Отредактировать ~/.ssh/config
nano ~/.ssh/config

# Добавить:
Host github.com
    User git
    HostName github.com
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes
    IdentitiesOnly yes

Host gitlab.com
    User git
    HostName gitlab.com
    IdentityFile ~/.ssh/id_ed25519

# Если SSH блокируется файрволом на порту 22
Host github.com
    User git
    HostName ssh.github.com
    Port 443
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes

# Теперь можно использовать
ssh -T git@github.com
# А не указывать порт каждый раз
```

## Тестирование SSH соединения

```bash
# Базовая проверка
ssh -T git@github.com
# Hi username! You've successfully authenticated...

# С более подробной информацией
ssh -vT git@github.com
# debug1: Remote protocol version 2.0, remote software version libssh_0.8.9
# debug1: match: OpenSSH_7.9 pat OpenSSH* compat 0x04000000
# ...
# Hi username! You've successfully authenticated...

# Проверить для GitLab
ssh -T git@gitlab.com

# Проверить для Bitbucket
ssh -T git@bitbucket.org
```

## Использование разных ключей для разных сервисов

Часто нужны разные ключи для разных сервисов (GitHub, GitLab, работа и личное):

```bash
# Создать несколько ключей
ssh-keygen -t ed25519 -C "personal@example.com" -f ~/.ssh/id_ed25519_personal
ssh-keygen -t ed25519 -C "work@company.com" -f ~/.ssh/id_ed25519_work

# Настроить в ~/.ssh/config
nano ~/.ssh/config

# Добавить:
Host github.com-personal
    User git
    HostName github.com
    IdentityFile ~/.ssh/id_ed25519_personal

Host github.com-work
    User git
    HostName github.com
    IdentityFile ~/.ssh/id_ed25519_work

Host gitlab.com
    User git
    HostName gitlab.com
    IdentityFile ~/.ssh/id_ed25519_personal

# Использовать в URL
git clone git@github.com-personal:username/personal-repo.git
git clone git@github.com-work:mycompany/work-repo.git
```

## Корпоративные файрволы и прокси

Иногда HTTPS блокируется или требует сертификат, SSH блокируется на порту 22:

```bash
# Если SSH блокируется на порту 22, GitHub поддерживает порт 443
Host github.com
    HostName ssh.github.com
    Port 443
    IdentityFile ~/.ssh/id_ed25519

ssh -T git@github.com
# Это должно сработать

# Если нужен корпоративный HTTPS прокси
git config --global http.proxy http://proxy.company.com:8080
git config --global https.proxy http://proxy.company.com:8080

# Или только для одного домена
git config --global http.https://github.com.proxy http://proxy.company.com:8080

# Если прокси требует аутентификацию
git config --global http.proxy http://user:password@proxy.company.com:8080

# Если нужно избежать прокси для localhost
git config --global http.noProxy localhost,127.0.0.1,*.company.com
```

## Сравнение скорости

На практике разница в скорости минимальна для большинства операций:

```bash
# Клонирование большого репо (HTTPS vs SSH)
# HTTPS: ~30 сек
# SSH:   ~28 сек
# Разница: 2 сека на крупный репо

# Для обычного использования (push/pull)
# Различия не заметны
```

## Часто задаваемые вопросы

**Какой протокол быстрее?** SSH немного быстрее, но разница незначительна для большинства случаев. Для крупных репозиториев разница может быть 5-10%.

**Какой более безопасен?** SSH считается более безопасным: ключи криптографически сильнее паролей, приватный ключ не передаётся по сети, не требует хранения длинного токена.

**Можно ли использовать оба протокола?** Да, разные репозитории могут использовать разные протоколы. Но для одного репо выбирайте один протокол.

**GitHub требует SSH или HTTPS?** Ни то, ни другое не требуется — выбирайте удобный вам. GitHub поддерживает оба протокола с одинаковой функциональностью.

**SSH блокируется файрволом — что делать?** GitHub поддерживает SSH через порт 443 (вместо обычного 22). Добавьте в `~/.ssh/config`: `Host github.com HostName ssh.github.com Port 443`.

**Где хранить токен безопасно?** Используйте credential helpers: `git config credential.helper cache` (временно в памяти) или `osxkeychain` (на macOS в защищённом хранилище).

## Заключение

Для начинающих рекомендуется HTTPS — меньше настройки. Для регулярной профессиональной работы — SSH: настроить один раз и забыть о паролях. Большинство опытных разработчиков используют SSH для личных компьютеров.

Если хотите настроить SSH — читайте [подробное руководство по настройке SSH для GitHub]({{< relref "nastrojka-ssh-github" >}}) или [общее руководство по подключению к Git-серверам по SSH]({{< relref "podklyuchenie-k-git-po-ssh" >}}).
