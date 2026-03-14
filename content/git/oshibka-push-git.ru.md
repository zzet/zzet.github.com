---
title: "Не удалось отправить в удалённый репозиторий: ошибка git push"
description: "Диагностика и решение ошибки «не удалось отправить в удалённый репозиторий». Причины: конфликты, права доступа, SSH. Пошаговые решения."
date: 2026-02-20
lastmod: 2026-02-20
draft: false
slug: "oshibka-push-git"
keywords: ["не удалось отправить в удалённый репозиторий", "не удалось отправить некоторые ссылки в git", "ошибка push git", "git push rejected", "не могу запушить git", "error push git"]
tags: ["git", "push", "troubleshooting"]
categories: ["git"]
---

`git push` завершился ошибкой. Это случается с каждым разработчиком. Причины могут быть разными: от конфликтов истории до проблем с SSH. Разберём каждую по порядку.

## Диагностика: чтение сообщения об ошибке

Git даёт информативные сообщения — важно их читать:

```
! [rejected]        main -> main (non-fast-forward)
error: failed to push some refs to 'https://github.com/user/repo.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. Integrate the remote changes before pushing again.
```

Здесь написано: удалённая ветка содержит коммиты, которых нет у вас локально. Решение — сначала получить их (`git pull`).

```
ERROR: Repository not found.
fatal: Could not read from remote repository.
```

Это проблема с доступом или неправильным URL.

```
remote: Permission to user/repo.git denied to your-username.
```

У вашего аккаунта нет прав на запись в этот репозиторий.

## Причина 1: удалённая ветка содержит новые изменения

Самая частая причина. Кто-то другой отправил коммиты пока вы работали.

```bash
# Получить обновления без слияния
git fetch origin

# Посмотреть что изменилось
git log HEAD..origin/main --oneline

# Вариант 1: слить через merge
git pull origin main
git push origin main

# Вариант 2: слить через rebase (чище история)
git pull --rebase origin main
git push origin main

# Если pull не требует конфликтов, push пройдёт успешно
```

## Причина 2: проблемы с SSH ключами

Если видите `Permission denied (publickey)`:

```bash
# Проверить что SSH ключ добавлен
ssh-add -l
# Если список пустой — добавить ключ:
ssh-add ~/.ssh/id_ed25519

# Проверить подключение к GitHub
ssh -T git@github.com
# Должно вывести: "Hi username! You've successfully authenticated..."

# Проверить права на файл ключа
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

Если ключа нет совсем — создать и добавить на GitHub/GitLab:

```bash
ssh-keygen -t ed25519 -C "email@example.com"
cat ~/.ssh/id_ed25519.pub  # скопировать и добавить в настройки GitHub
ssh -T git@github.com     # проверить
```

## Причина 3: неправильный URL remote

```bash
# Проверить URL
git remote -v

# Если URL неправильный — исправить
git remote set-url origin https://github.com/correct-user/correct-repo.git

# Переключить с HTTPS на SSH (если SSH ключ настроен)
git remote set-url origin git@github.com:username/repo.git

# Проверить
git remote -v
git push origin main
```

## Причина 4: нет прав на запись в репозиторий

Если репозиторий не ваш и вы не добавлены как contributor:

```bash
# Проверить кем вы аутентифицированы
ssh -T git@github.com
# Вывод: Hi your-username!

# Если username не тот — нужно настроить правильный ключ или токен
```

Для HTTPS можно попасть в ситуацию, когда сохранён старый токен:

```bash
# Очистить сохранённые учётные данные (macOS)
git credential-osxkeychain erase
host=github.com
protocol=https
# Enter (пустая строка)

# Или через git config
git config --global credential.helper store
# Затем выполнить push и ввести новый токен
```

## Причина 5: non-fast-forward (история расходится)

Если удалённая ветка имеет другую историю (например, после rebase или reset с force push):

```bash
# Проверить состояние
git log --oneline -5
git log --oneline origin/main -5

# Попробовать интегрировать изменения
git pull --rebase origin main

# Если конфликты — разрешить и продолжить
git add <resolved-files>
git rebase --continue

# Затем push
git push origin main
```

Если точно уверены, что ваша история правильная (например, сознательно делаете rebase):

```bash
# Безопасный force push (проверяет что никто не push-нул пока вы работали)
git push --force-with-lease origin main

# НЕ используйте git push --force в shared ветках!
# Это перезапишет чужие коммиты без предупреждения
```

## Причина 6: защита ветки (branch protection rules)

На GitHub/GitLab можно настроить защиту main/master:
- Прямой push запрещён — нужен pull request
- Требуется code review
- Ветка заморожена

```bash
# Создать отдельную ветку и открыть PR
git checkout -b feature/my-fix
# ... работа ...
git push origin feature/my-fix
# Затем открыть pull request на GitHub
```

## Пошаговая диагностика

Если не ясно в чём проблема:

```bash
# 1. Проверить статус
git status

# 2. Убедиться что мы на правильной ветке
git branch

# 3. Проверить remote
git remote -v

# 4. Проверить SSH подключение
ssh -T git@github.com

# 5. Загрузить состояние сервера
git fetch origin

# 6. Сравнить локальную и удалённую ветку
git log HEAD..origin/main --oneline  # коммиты на сервере
git log origin/main..HEAD --oneline  # наши локальные коммиты

# 7. Слить изменения с сервера
git pull --rebase origin main

# 8. Повторить push
git push origin main
```

## Лучшие практики для предотвращения ошибок

`git pull` или `git fetch` перед началом работы — это сэкономит время. Работайте в feature-ветках, а не в main. Используйте `--force-with-lease` вместо `--force` если необходим force push.

## Часто задаваемые вопросы

**Что означает "non-fast-forward"?** Это значит, что удалённая ветка содержит коммиты, которых нет у вас. Git не может просто добавить ваши коммиты поверх — нужно сначала интегрировать чужие изменения через `git pull`.

**Когда безопасно использовать force push?** Только в ваших личных ветках, которые никто другой не использует. Никогда — в main/master или shared ветках. Всегда используйте `--force-with-lease` вместо `--force`.

**Как добавить SSH ключ в ssh-agent?** `ssh-add ~/.ssh/id_ed25519`. Чтобы не добавлять при каждом перезапуске, добавьте в `~/.ssh/config`:
```
Host github.com
  AddKeysToAgent yes
  IdentityFile ~/.ssh/id_ed25519
```

**Почему Git говорит, что мне нужно pull перед push?** На сервере есть коммиты, которых нет у вас. Git защищает от случайного удаления чужих коммитов — нужно сначала интегрировать их.

## Заключение

Большинство ошибок push решаются через `git pull --rebase origin main`. Проблемы с доступом — через проверку SSH ключей и URL. Перед push всегда полезно выполнить [git fetch]({{< relref "git-fetch" >}}) и проверить состояние через [git status]({{< relref "git-status" >}}). О конфликтах при merge — [конфликты git merge]({{< relref "konflikty-git-merge" >}}).
