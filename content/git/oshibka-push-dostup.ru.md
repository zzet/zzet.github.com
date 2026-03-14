---
title: "Ошибка push: нет доступа к репозиторию GitHub — причины и решения"
description: "Ошибки при git push на GitHub: Permission denied, Repository not found, Authentication failed. Диагностика и решение проблем с доступом."
date: 2026-02-19
lastmod: 2026-02-19
draft: false
slug: "oshibka-push-dostup"
keywords: ["git push ошибка доступа", "permission denied github", "git push authentication failed", "github push permission denied", "403 git push github", "authentication failed git push"]
tags: ["git", "github", "beginner"]
categories: ["git"]
---

Ошибки при `git push` на GitHub возникают по нескольким причинам: неправильно настроенный SSH ключ, неверный URL репозитория, истёкший токен или недостаточные права. В большинстве случаев проблема диагностируется за несколько шагов.

## Ошибка: Permission denied (publickey)

Самая частая ошибка при работе через SSH:

```bash
git push origin main
# git@github.com: Permission denied (publickey).
# fatal: Could not read from remote repository.

# Диагностика:
ssh -T git@github.com
# git@github.com: Permission denied (publickey).
```

**Причина:** SSH ключ не добавлен в GitHub аккаунт или не загружен в ssh-agent.

**Решение:**

```bash
# Шаг 1: Проверить наличие SSH ключей
ls -la ~/.ssh/
# id_ed25519, id_ed25519.pub — хорошо
# (пусто) — нужно создать ключ

# Шаг 2: Создать ключ если нет
ssh-keygen -t ed25519 -C "your@email.com"

# Шаг 3: Добавить ключ в ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Шаг 4: Скопировать публичный ключ
cat ~/.ssh/id_ed25519.pub
# Скопировать вывод

# Шаг 5: Добавить ключ на GitHub
# GitHub → Settings → SSH and GPG keys → New SSH key
# Вставить скопированный ключ

# Шаг 6: Проверить
ssh -T git@github.com
# Hi username! You've successfully authenticated, but GitHub does
# not provide shell access.
```

## Ошибка: Authentication failed

При работе через HTTPS:

```bash
git push origin main
# remote: Invalid username or password.
# fatal: Authentication failed for 'https://github.com/user/repo.git'
```

**Причина:** GitHub не принимает пароли с 2021 года. Нужен Personal Access Token (PAT).

**Решение:**

```bash
# Шаг 1: Создать PAT на GitHub
# GitHub → Settings → Developer settings → Personal access tokens
# → Tokens (classic) → Generate new token
# Выбрать scope: repo (полный доступ к репозиториям)

# Шаг 2: Использовать токен вместо пароля
git push origin main
# Username: ваш_логин
# Password: ghp_xxxxxxxxxxxx  ← токен, не пароль

# Шаг 3: Кешировать токен (не вводить каждый раз)
git config --global credential.helper store
# После следующего push токен сохранится в ~/.git-credentials

# Лучший способ — системное хранилище
git config --global credential.helper osxkeychain  # macOS
git config --global credential.helper manager       # Windows
```

## Ошибка: Repository not found

```bash
git push origin main
# remote: Repository not found.
# fatal: repository 'https://github.com/user/project.git/' not found
```

**Причины и решения:**

```bash
# Проверить URL
git remote -v
# origin  https://github.com/user/project.git (fetch)

# Частые ошибки в URL:
# ✗ https://github.com/User/Project.git (неверный регистр)
# ✗ https://github.com/user/projet.git (опечатка)
# ✗ https://github.com/otheruser/project.git (чужой репозиторий)

# Исправить URL
git remote set-url origin https://github.com/correct-user/correct-repo.git

# Убедиться что репозиторий существует
# Открыть URL в браузере:
# https://github.com/user/project

# Если репозиторий существует но не найден:
# Возможно, нужно добавить токен в URL
git remote set-url origin https://TOKEN@github.com/user/project.git
```

## Ошибка: нет прав на push

```bash
git push origin main
# remote: error: GH006: Protected branch update failed for refs/heads/main.
# remote: error: Required status checks have not passed.

# Или:
# remote: Permission to user/project.git denied to youruser.
# fatal: unable to access 'https://github.com/user/project.git/':
#   The requested URL returned error: 403
```

**Решения:**

```bash
# Случай 1: Ветка защищена (branch protection)
# Нужно создать pull request вместо прямого push
git checkout -b feature/my-changes
git push origin feature/my-changes
# Затем создать PR на GitHub

# Случай 2: Нет прав на репозиторий
# Попросить owner добавить вас как collaborator:
# Repository → Settings → Collaborators → Add people

# Случай 3: Нужен fork workflow
# Сделать форк → клонировать форк → push в форк → PR в оригинальный
```

## Ошибка: ключ используется другим аккаунтом

```bash
ssh -T git@github.com
# Hi OTHER_USER! You've successfully authenticated...
# ← показывает не тот аккаунт!
```

**Решение для нескольких GitHub аккаунтов:**

```bash
# Создать отдельный ключ для каждого аккаунта
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_work -C "work@company.com"
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_personal -C "personal@gmail.com"

# ~/.ssh/config
cat >> ~/.ssh/config << 'EOF'
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
EOF

# Использовать алиасы
git remote set-url origin git@github-work:company/project.git
git remote set-url origin git@github-personal:myuser/project.git

# Проверить аккаунты
ssh -T git@github-work
ssh -T git@github-personal
```

## Ошибка: large files (100 МБ лимит)

```bash
git push origin main
# remote: error: GH001: Large files detected.
# remote: error: File bigfile.zip is 150.00 MB; this exceeds
#   GitHub's file size limit of 100.00 MB
```

**Решение:**

```bash
# Удалить большой файл из истории
git filter-repo --path bigfile.zip --invert-paths

# Или использовать Git LFS для больших файлов
git lfs install
git lfs track "*.zip"
git add .gitattributes
git add bigfile.zip
git commit -m "Add bigfile via LFS"
git push origin main
```

## Общая диагностика

```bash
# Полная диагностика SSH подключения
ssh -vT git@github.com 2>&1

# Проверить какой ключ используется
ssh -vT git@github.com 2>&1 | grep "Offering public key"

# Проверить remote URL
git remote -v

# Проверить права на репозиторий через API
curl -H "Authorization: token YOUR_TOKEN" \
  https://api.github.com/repos/user/project

# Включить трассировку git
GIT_TRACE=1 GIT_SSH_COMMAND="ssh -v" git push 2>&1
```

## Часто задаваемые вопросы

**После смены пароля GitHub перестал работать git push?** GitHub не использует пароли для Git операций. Используйте Personal Access Token (PAT) вместо пароля, или SSH ключи.

**Как добавить SSH ключ если потерял доступ к аккаунту?** Восстановите доступ через интерфейс GitHub (двухфакторная аутентификация, резервные коды). После восстановления — добавьте новый SSH ключ.

**git push работал, но перестал после ротации токена?** Обновите токен в системном хранилище паролей или в `~/.git-credentials`. Выполните `git credential reject https://github.com` и повторите push — система запросит новые учётные данные.

**Можно ли сделать push без SSH и без токена?** Только через GitHub CLI (`gh`) с OAuth аутентификацией. `gh auth login` настроит учётные данные автоматически.

**Почему Permission denied только на одном компьютере?** SSH ключ этого компьютера не добавлен в GitHub аккаунт. На другом компьютере другой ключ, который добавлен.

## Заключение

Большинство ошибок push на GitHub сводятся к трём причинам: неправильный SSH ключ, устаревший токен, недостаточные права. Диагностируйте с помощью `ssh -T git@github.com` (SSH) или проверки токена (HTTPS). Настройте системное хранилище учётных данных чтобы не вводить токен при каждой операции. Аналогичные ошибки в GitLab рассмотрены в [отдельной статье]({{< relref "oshibka-push-dostup-gitlab" >}}).
