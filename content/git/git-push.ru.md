---
title: "git push: как отправить код в удалённый репозиторий Git"
description: "Полное руководство по команде git push. Примеры: push в origin, push конкретной ветки, force push. Решение типичных ошибок."
date: 2026-01-23
lastmod: 2026-01-23
draft: false
slug: "git-push"
keywords: ["git push origin что делает", "git push origin develop", "отправить изменения git", "как загрузить код в git"]
tags: ["git", "remote", "beginner"]
categories: ["git"]
aliases: []
---

Когда вы создали коммиты локально, они существуют только на вашем компьютере. Чтобы поделиться кодом с командой или сохранить его в облаке (GitHub, GitLab, Bitbucket), нужна команда `git push`. Это один из важнейших шагов в ежедневной работе разработчика.

## Что происходит при git push

`git push` отправляет локальные коммиты в удалённый репозиторий. Git сравнивает историю коммитов локальной и удалённой ветки, находит коммиты, которых нет на сервере, и загружает их.

```
Локально:  A ← B ← C ← D (новые коммиты C и D)
Удалённо:  A ← B

После push:
Удалённо:  A ← B ← C ← D
```

## Базовый синтаксис

```bash
git push [remote] [branch]
```

- `remote` — имя удалённого репозитория (обычно `origin`)
- `branch` — ветка для отправки

## Основные варианты использования

### Простой push (если ветка отслеживается)

Если ветка уже настроена на отслеживание удалённой:
```bash
git push
```

### Явный push в конкретную ветку

```bash
git push origin main
git push origin develop
git push origin feature/auth
```

### Первый push новой ветки с настройкой отслеживания

При первом push новой ветки используйте флаг `-u` (или `--set-upstream`):

```bash
git push -u origin feature/new-dashboard
# или
git push --set-upstream origin feature/new-dashboard
```

Флаг `-u` устанавливает связь между локальной и удалённой веткой. После этого в этой ветке достаточно писать просто `git push`.

### Push всех веток

```bash
git push --all origin
```

## Что такое origin

`origin` — это просто имя (псевдоним) вашего удалённого репозитория. Когда вы клонируете репозиторий через `git clone`, Git автоматически создаёт `origin`, указывающий на исходный URL.

```bash
# Посмотреть что такое origin
git remote -v
# origin  git@github.com:username/project.git (fetch)
# origin  git@github.com:username/project.git (push)
```

У вас может быть несколько remote: `origin`, `upstream`, `backup` и т.д.

## Проверка перед push

Хорошая практика — проверить, что именно будет отправлено:

```bash
# Посмотреть статус
git status

# Посмотреть коммиты, которых ещё нет на сервере
git log origin/main..HEAD --oneline

# Или разницу
git diff origin/main
```

## Удаление удалённой ветки

```bash
git push origin --delete feature/old-branch
# Или устаревший синтаксис:
git push origin :feature/old-branch
```

## Push тегов

```bash
# Отправить конкретный тег
git push origin v1.2.0

# Отправить все теги
git push origin --tags
```

## Force push: переписываем историю

В редких случаях нужно принудительно перезаписать удалённую ветку:

```bash
git push -f origin main
# или
git push --force origin main
```

**Важно:** force push переписывает историю на сервере. Если кто-то уже скачал эти коммиты — у него возникнут конфликты. Никогда не делайте force push в общие ветки (`main`, `develop`).

### Более безопасный вариант: --force-with-lease

```bash
git push --force-with-lease origin feature/my-branch
```

Этот вариант откажется от push, если удалённая ветка изменилась с тех пор, как вы последний раз делали `git fetch`. Это защищает от случайного затирания чужих коммитов.

Когда force push допустим:
- В личной feature-ветке, над которой работаете только вы
- После `git rebase` или `git commit --amend` в личной ветке
- Исправление случайно закоммиченных секретов (после смены ключей!)

## Обработка ошибок

### Rejected: outdated

```
! [rejected]        main -> main (fetch first)
error: failed to push some refs
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
```

Причина: на сервере есть коммиты, которых нет у вас. Решение:
```bash
git pull origin main   # Получить обновления и слить
git push origin main   # Теперь push пройдёт
```

### Rejected: non-fast-forward

Похожая ситуация, история разошлась. Решение то же: `git pull`.

### Permission denied (publickey)

```
git@github.com: Permission denied (publickey).
```

Причина: проблема с SSH-ключами. Решение: проверьте настройку SSH:
```bash
ssh -T git@github.com
```

### Repository not found

```
ERROR: Repository not found.
```

Причина: неправильный URL или нет доступа. Проверьте:
```bash
git remote -v  # Проверить URL
```

## Dry run: проверка без реального push

```bash
git push --dry-run origin main
# Показывает, что будет сделано, не отправляя ничего реально
```

## Лучшие практики

**Коммитьте часто с небольшими изменениями.** Маленькие коммиты проще ревьюить и откатывать.

**Делайте pull перед push.** `git pull` перед `git push` снижает вероятность конфликтов.

**Проверяйте перед push.** `git diff origin/main` или `git log origin/main..HEAD` покажут что будет отправлено.

**Не делайте force push в общие ветки.** `main`, `develop` — это общие ветки. Force push здесь сломает работу всей команды.

**Используйте `--force-with-lease` вместо `--force`.** Это безопаснее.

## Полный пример рабочего цикла

```bash
# 1. Обновить локальную копию
git switch main
git pull origin main

# 2. Создать ветку для задачи
git switch -c feature/user-profile

# 3. Работать и коммитить
echo "profile code" > profile.js
git add profile.js
git commit -m "feat: add user profile page"

# 4. Проверить что будет запушено
git log origin/main..HEAD --oneline

# 5. Первый push с установкой отслеживания
git push -u origin feature/user-profile

# 6. Дальнейшие push'и — просто
git push

# 7. Создать Pull Request на GitHub (через веб-интерфейс или CLI)
# gh pr create --title "Add user profile" --body "..."
```

## Часто задаваемые вопросы

**Что означает `origin` в команде `git push origin main`?** `origin` — это имя (псевдоним) вашего удалённого репозитория. Обычно это URL на GitHub, GitLab или другой платформе. Можно посмотреть через `git remote -v`.

**Чем отличается `git push` от `git push origin main`?** `git push` использует настроенное отслеживание (upstream). `git push origin main` явно указывает куда и что отправить. Для первого push новой ветки нужно явно указать remote и ветку (или использовать `-u`).

**Что делает флаг `-u` в `git push -u origin main`?** Устанавливает связь между локальной и удалённой веткой. После этого `git push` и `git pull` будут знать, куда отправлять и откуда получать, без дополнительных аргументов.

**Когда нужен force push и почему он опасен?** Force push нужен после rebase или amend в личной ветке. Он переписывает историю на сервере, что сломает работу всех, кто уже скачал эти коммиты. Никогда не делайте force push в общие ветки.

**Как отменить случайный push?** Если никто ещё не скачал коммиты — `git revert HEAD` создаст отменяющий коммит и запушьте его. Если нужно убрать коммит полностью — `git reset` + force push в личной ветке.

**Можно ли отправить определённый коммит, не отправляя остальные?** Да: `git push origin <хеш>:main` отправит до указанного коммита. Но обычно проще работать с ветками.

## Заключение

`git push` — финальный шаг в рабочем цикле разработчика. Освойте флаги `-u` для первого push ветки и `--force-with-lease` для безопасного force push. Всегда делайте `git pull` перед push, чтобы избежать rejected-ошибок.

Связанные команды: «[git pull: как получить обновления]({{< relref "git-pull" >}})» и «[git commit: полное руководство]({{< relref "git-commit" >}})».
