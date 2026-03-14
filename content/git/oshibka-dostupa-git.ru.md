---
title: "Ошибка доступа в Git: недостаточно прав к репозиторию"
description: "Решение ошибки «удостоверьтесь что у вас есть необходимые права». SSH ключи, конфигурация URL, проверка доступа. Пошаговое руководство."
date: 2026-02-18
lastmod: 2026-02-18
draft: false
slug: "oshibka-dostupa-git"
keywords: ["удостоверьтесь что у вас есть необходимые права доступа и репозиторий существует", "ошибка доступа git", "permission denied git", "git repository does not exist", "git permission denied публичный ключ", "нет прав git"]
tags: ["git", "ssh", "troubleshooting"]
categories: ["git"]
---

`fatal: Could not read from remote repository. Please make sure you have the correct access rights and the repository exists.` — эта ошибка появляется при клонировании, push или fetch. Причин несколько, и диагностика начинается с выяснения — это проблема с аутентификацией или с самим URL?

## Понимание сообщения об ошибке

Git не может получить доступ к репозиторию по одной из причин:
- Неправильный SSH ключ или он не добавлен
- Неправильный URL репозитория
- У вас нет прав на доступ (репозиторий приватный)
- Репозиторий не существует

Первый шаг — определить тип проблемы.

## Проверка SSH подключения

Начните с проверки SSH соединения:

```bash
# Проверить подключение к GitHub
ssh -T git@github.com
# Успех: "Hi username! You've successfully authenticated, but GitHub does not provide shell access."
# Ошибка: "Permission denied (publickey)."

# Проверить подключение к GitLab
ssh -T git@gitlab.com
# Успех: "Welcome to GitLab, @username!"

# Проверить список загруженных ключей
ssh-add -l
# Если "The agent has no identities" — ключи не добавлены
```

Если SSH проверка выдаёт ошибку — проблема в SSH конфигурации.

## Причина 1: SSH ключ не добавлен в ssh-agent

```bash
# Добавить SSH ключ в агент
ssh-add ~/.ssh/id_ed25519
# или
ssh-add ~/.ssh/id_rsa

# Проверить что ключ добавлен
ssh-add -l

# Повторить проверку соединения
ssh -T git@github.com
```

На macOS ключ сбрасывается после перезагрузки. Для автоматического добавления добавьте в `~/.ssh/config`:

```
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

## Причина 2: Публичный ключ не добавлен на GitHub/GitLab

SSH ключ работает в паре: приватный ключ у вас, публичный — на сервере. Если публичный ключ не добавлен:

```bash
# Посмотреть публичный ключ
cat ~/.ssh/id_ed25519.pub
# Или RSA:
cat ~/.ssh/id_rsa.pub

# Скопировать содержимое и добавить на GitHub:
# Settings → SSH and GPG keys → New SSH key → вставить
```

Затем проверить:
```bash
ssh -T git@github.com
```

## Причина 3: Нет SSH ключей — создать новые

```bash
# Создать новый Ed25519 ключ (рекомендуется)
ssh-keygen -t ed25519 -C "your-email@example.com"

# Или RSA (если Ed25519 не поддерживается)
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"

# Сохранить в стандартном месте (нажать Enter)
# Ввести passphrase или оставить пустым

# Добавить в ssh-agent
ssh-add ~/.ssh/id_ed25519

# Посмотреть публичный ключ (добавить на GitHub/GitLab)
cat ~/.ssh/id_ed25519.pub
```

## Причина 4: Используется HTTPS, но нет токена

Если URL в формате `https://github.com/...`, Git попросит логин/пароль. GitHub убрал поддержку паролей — нужен Personal Access Token:

```bash
# Посмотреть текущий URL
git remote -v

# Если HTTPS и нет сохранённого токена — при push/fetch запросят учётные данные
# Используйте токен вместо пароля

# Сохранить учётные данные
git config --global credential.helper store
# После первого ввода — сохранятся автоматически

# Или переключиться на SSH
git remote set-url origin git@github.com:username/repo.git
```

## Причина 5: Неправильный URL

```bash
# Проверить URL
git remote -v

# Распространённые ошибки:
# git@github.com:wrong-user/repo.git  — неправильный пользователь
# git@github.com:user/wrong-repo.git  — неправильное имя репо
# https://github.com/user/repo        — нет .git на конце

# Исправить URL
git remote set-url origin git@github.com:correct-user/correct-repo.git

# Проверить
git remote -v
```

## Причина 6: Нет прав на репозиторий

Если репозиторий приватный, и вы не добавлены как collaborator:

```bash
# Проверить от чьего имени вы аутентифицированы
ssh -T git@github.com
# "Hi your-username!"

# Если username неправильный — используется другой ключ
# Проверить все ключи
ssh-add -l

# Указать явно какой ключ использовать в ~/.ssh/config:
Host github.com
  IdentityFile ~/.ssh/correct-key
```

Если username правильный, но доступа нет — обратитесь к владельцу репозитория для добавления прав.

## Пошаговая диагностика

```bash
# 1. Проверить URL remote
git remote -v

# 2. Проверить SSH соединение
ssh -T git@github.com

# 3. Если ошибка — проверить ключи в агенте
ssh-add -l

# 4. Добавить ключ если не добавлен
ssh-add ~/.ssh/id_ed25519

# 5. Если ключей нет — создать и добавить на GitHub
ssh-keygen -t ed25519 -C "email@example.com"
cat ~/.ssh/id_ed25519.pub
# Добавить на GitHub в Settings → SSH keys

# 6. Снова проверить
ssh -T git@github.com

# 7. Повторить git push/clone
```

## Отладка SSH соединения

Если стандартная проверка не даёт ясного ответа:

```bash
# Подробный лог SSH соединения
ssh -vvv git@github.com

# Поможет увидеть:
# - Какие ключи пробуются
# - Где именно происходит ошибка
# - Проблемы с правами доступа на файлы
```

## Часто задаваемые вопросы

**Как добавить SSH ключ в ssh-agent?** `ssh-add ~/.ssh/id_ed25519`. На macOS: `ssh-add --apple-use-keychain ~/.ssh/id_ed25519` для постоянного сохранения в Keychain.

**Где находится мой публичный SSH ключ?** `~/.ssh/id_ed25519.pub` или `~/.ssh/id_rsa.pub`. Просмотреть: `cat ~/.ssh/id_ed25519.pub`.

**Чем отличается SSH от HTTPS для Git?** SSH использует ключи (не нужен пароль при каждой операции), надёжнее для частого использования. HTTPS использует логин/токен, проще настроить без SSH.

**Как добавить SSH ключ на GitHub?** GitHub → Settings → SSH and GPG keys → New SSH key → вставить содержимое `~/.ssh/id_ed25519.pub`.

**Почему мой SSH ключ не работает на новой машине?** Ключи не переносятся автоматически. На новой машине нужно создать новый ключ и добавить его публичную часть на GitHub/GitLab.

## Заключение

Большинство ошибок доступа решаются через: `ssh-add ~/.ssh/id_key` + проверку `ssh -T git@github.com` + добавление публичного ключа на GitHub, если нужно. Для первоначальной настройки SSH — [подключение к Git по SSH]({{< relref "podklyuchenie-k-git-po-ssh" >}}). Для исправления URL remote — [git remote add]({{< relref "git-remote-add" >}}).
