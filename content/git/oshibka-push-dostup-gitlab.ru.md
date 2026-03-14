---
title: "Ошибка push на GitLab: доступ запрещён, диагностика и решения"
description: "Ошибки при git push на GitLab: Permission denied, Repository not found, доступ запрещён. GitLab-специфичные причины: роли, protected branches, deploy tokens."
date: 2026-02-19
lastmod: 2026-02-19
draft: false
slug: "oshibka-push-dostup-gitlab"
keywords: ["git push gitlab ошибка", "gitlab permission denied", "gitlab push access denied", "ошибка push gitlab", "403 gitlab push"]
tags: ["git", "gitlab", "beginner"]
categories: ["git"]
---

Ошибки при `git push` на GitLab имеют специфику по сравнению с GitHub: GitLab использует свою систему ролей, настройки protected branches и deploy tokens. Разбираем типичные ошибки и способы их решения.

## Ошибка: Permission denied (publickey)

```bash
git push origin main
# git@gitlab.com: Permission denied (publickey).
# fatal: Could not read from remote repository.
```

**Диагностика:**

```bash
# Проверить SSH подключение к GitLab
ssh -T git@gitlab.com
# Welcome to GitLab, @username!  ← успех
# Permission denied (publickey). ← проблема

# Подробная диагностика
ssh -vT git@gitlab.com 2>&1 | head -30
```

**Решение:**

```bash
# Создать SSH ключ (если нет)
ssh-keygen -t ed25519 -C "your@email.com"

# Добавить ключ в ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Скопировать публичный ключ
cat ~/.ssh/id_ed25519.pub

# Добавить в GitLab
# User Settings → SSH Keys → Add new key
# Вставить ключ, указать срок действия, сохранить

# Проверить
ssh -T git@gitlab.com
# Welcome to GitLab, @username!
```

## Ошибка: доступ запрещён из-за роли

GitLab использует строгую систему ролей. Если у вас роль Reporter или Guest — push заблокирован:

```bash
git push origin feature/my-changes
# remote: GitLab: You are not allowed to push code to this project.
# ! [remote rejected] feature/my-changes -> feature/my-changes (pre-receive hook declined)

# Или:
# remote: GitLab: You are not allowed to push code to protected branches on this project.
```

**Причины:**

```
Guest (10)    — только просмотр issues
Reporter (20) — чтение кода, нет push
Developer (30) — push в незащищённые ветки ✓
Maintainer (40) — push в защищённые ветки ✓
Owner (50)    — все операции ✓
```

**Решение:**

```bash
# Запросить повышение роли у Maintainer/Owner:
# Project → Project Settings → Members → ваш аккаунт → изменить роль на Developer

# Если вы Maintainer: изменить роль другому пользователю
# Project → Manage → Members → выбрать пользователя → изменить роль
```

## Ошибка: protected branch

```bash
git push origin main
# remote: GitLab: You are not allowed to push code to protected branches on this project.
```

В GitLab ветки `main`/`master` защищены по умолчанию:

```bash
# Решение 1: Push в новую ветку и создать MR
git checkout -b feature/my-fix
git push origin feature/my-fix
# Затем создать Merge Request в GitLab

# Решение 2: Изменить настройки защиты (если вы Maintainer/Owner)
# Project → Settings → Repository → Protected Branches
# Найти main → изменить "Allowed to push" на Developer

# Решение 3: Если нужен push с force
git push --force-with-lease origin main
# Только если Maintainer и настройка "Allowed to force push" включена
```

## Ошибка: аутентификация через HTTPS

```bash
git push https://gitlab.com/user/project.git
# remote: HTTP Basic: Access denied. The provided password or token is incorrect.
# fatal: Authentication failed
```

**Решение:**

```bash
# Создать Personal Access Token в GitLab
# User Settings → Access Tokens → Add new token
# Scope: api (или write_repository для минимальных прав)

# Использовать токен
git push https://gitlab.com/user/project.git
# Username: ваш_логин
# Password: glpat-xxxxxxxxxxxx  ← токен (начинается с glpat-)

# Или вставить токен в URL
git remote set-url origin https://oauth2:TOKEN@gitlab.com/user/project.git

# Кешировать учётные данные
git config --global credential.helper store
```

## Deploy Tokens и Deploy Keys

GitLab предлагает специальные токены для CI/CD:

```bash
# Deploy Token: для CI/CD пайплайнов (чтение/запись)
# Project → Settings → Repository → Deploy tokens
# Создать токен с write_repository scope

# Использование deploy token
git clone https://deploy-token-user:TOKEN@gitlab.com/user/project.git

# Deploy Key: SSH ключ только для чтения (для деплоя)
# Project → Settings → Repository → Deploy keys
# Добавить публичный SSH ключ

# Deploy keys по умолчанию read-only
# Для push включить: "Grant write permissions to this key"
```

## Ошибка: namespace или проект не найден

```bash
git push origin main
# remote: The project you were looking for could not be found or you
#   don't have permission to view it.
# fatal: repository 'https://gitlab.com/user/project.git/' not found
```

```bash
# Проверить URL
git remote -v

# Частые причины:
# 1. Проект перенесён или переименован
git remote set-url origin git@gitlab.com:new-group/new-name.git

# 2. Нет доступа к приватному проекту
# Запросить доступ или убедиться что вошли в правильный аккаунт

# 3. Проект в группе
git remote set-url origin git@gitlab.com:group-name/project.git

# 4. Нет доступа к репозиторию в группе
# Group → Manage → Members → добавить себя
```

## Self-hosted GitLab: специфичные ошибки

```bash
# Ошибка: SSL certificate problem
git push
# SSL certificate problem: self signed certificate

# Решение: добавить CA сертификат
git config --global http.sslCAInfo /path/to/ca.crt
# Или добавить хост
git config --global http.https://gitlab.company.com.sslVerify false

# Ошибка: нестандартный SSH порт
# Если GitLab использует порт не 22
git remote set-url origin ssh://git@gitlab.company.com:2222/user/project.git

# ~/.ssh/config
cat >> ~/.ssh/config << 'EOF'
Host gitlab.company.com
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_work
EOF

# Ошибка: timeout при подключении
# Возможно, порт заблокирован. Попробовать HTTPS
git remote set-url origin https://gitlab.company.com/user/project.git
```

## Push Rules в GitLab

GitLab позволяет настроить правила для push:

```bash
# Если push отклонён из-за push rules:
# remote: GitLab: Commit message does not follow the pattern.
# remote: GitLab: Author's commit email does not follow the pattern.

# Проверить правила:
# Project → Settings → Repository → Push rules

# Типичные правила:
# - Commit message должен соответствовать паттерну (Jira ID, Conventional Commits)
# - Email автора должен быть корпоративным
# - Запрет push secrets (токены, пароли в коде)
# - Запрет force push
```

## Полная диагностика

```bash
# Шаг 1: Проверить SSH
ssh -T git@gitlab.com

# Шаг 2: Проверить remote URL
git remote -v

# Шаг 3: Проверить роль на проекте
# GitLab UI: Project → Manage → Members → найти себя

# Шаг 4: Проверить защиту ветки
# GitLab UI: Project → Settings → Repository → Protected Branches

# Шаг 5: Включить трассировку
GIT_SSH_COMMAND="ssh -v" git push 2>&1

# Шаг 6: Проверить через API
curl --header "PRIVATE-TOKEN: your_token" \
  "https://gitlab.com/api/v4/projects/user%2Fproject/members/all"
```

## Часто задаваемые вопросы

**Почему Developer не может делать push в main?** По умолчанию в GitLab ветка `main` защищена: push разрешён только Maintainer и Owner. Developer может push только в незащищённые ветки или после изменения настроек protected branches.

**Как сделать push в форкнутый репозиторий?** Используйте URL своего форка, не оригинального репозитория. `git remote set-url origin git@gitlab.com:your-user/forked-project.git`.

**Personal Access Token или Deploy Token — что выбрать?** PAT привязан к пользователю — если пользователь уходит, токен становится недействительным. Deploy Token привязан к проекту — используйте его для CI/CD и автоматизации.

**Ошибка 403 при push через HTTPS?** Убедитесь что токен имеет нужный scope (`write_repository` или `api`). Токены с истёкшим сроком действия тоже дают 403.

**Как найти IP GitLab сервера если нет доступа к интерфейсу?** Для облачного gitlab.com — `nslookup gitlab.com`. Для self-hosted — обратитесь к администратору.

## Заключение

Ошибки push на GitLab чаще всего связаны с ролями (нужен Developer+), protected branches (создайте MR) или SSH ключами (добавьте в User Settings). Проверьте роль пользователя через Project → Members. Аналогичные ошибки на GitHub описаны в [статье про GitHub push]({{< relref "oshibka-push-dostup" >}}).
