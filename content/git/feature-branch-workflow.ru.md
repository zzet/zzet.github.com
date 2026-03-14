---
title: "Feature Branch Workflow: отдельная ветка для каждой задачи"
description: "Руководство по Feature Branch Workflow. Создание feature веток, Pull Request процесс, слияние, синхронизация с main. Лучшие практики именования."
date: 2026-01-04
lastmod: 2026-01-04
draft: false
slug: "feature-branch-workflow"
keywords: ["feature branch workflow", "feature ветка git", "ветка для каждой задачи", "feature branch git", "ветка для фичи git", "git flow feature branch"]
tags: ["git", "workflow", "beginner"]
categories: ["git"]
---

Feature Branch Workflow — базовый принцип: каждая новая функция или исправление разрабатывается в отдельной ветке. Это основа большинства Git моделей ветвления (GitHub Flow, Gitflow). Позволяет разрабатывать параллельно и интегрировать через code review.

## Принципы

```
1. main всегда стабилен — нет прямых коммитов в main
2. Каждая задача — своя ветка от main
3. Ветка называется по задаче: feature/user-auth, fix/login-bug
4. Регулярные коммиты в ветку
5. Pull Request → code review → merge
6. Удалить ветку после merge
```

## Создание feature ветки

```bash
# Обновить main
git switch main
git pull origin main

# Создать ветку для задачи
git switch -c feature/user-authentication
# или
git switch -c fix/login-redirect-bug
git switch -c docs/api-documentation
git switch -c chore/update-dependencies
```

## Именование веток

```
Рекомендуемые паттерны:
feature/<название>      — новая функциональность
fix/<что-исправляем>    — исправление бага
hotfix/<что-срочно>     — срочное исправление
docs/<тема>             — документация
chore/<задача>          — инфраструктура, зависимости
refactor/<что>          — рефакторинг
test/<что-тестируем>    — тесты

Конкретные примеры:
feature/dark-mode
feature/user-profile-editing
fix/null-pointer-dashboard
fix/incorrect-date-formatting
hotfix/security-patch-xss
docs/api-authentication
chore/upgrade-to-node-20
refactor/auth-service
```

## Разработка в ветке

```bash
# Регулярные коммиты
git commit -am "feat: add login form UI"
git commit -am "feat: implement JWT authentication"
git commit -am "test: add auth service tests"
git commit -am "fix: handle token expiration"

# Регулярно пушить (backup + CI feedback)
git push origin feature/user-authentication
```

## Синхронизация с main

При долгой разработке main уходит вперёд. Нужно синхронизироваться:

```bash
# Вариант 1: rebase (чистая история)
git fetch origin
git rebase origin/main

# Разрешить конфликты если есть, затем:
git rebase --continue

# Форс-пуш после rebase (ветка переписана)
git push --force-with-lease origin feature/user-authentication

# Вариант 2: merge (сохраняет историю)
git merge origin/main
git push origin feature/user-authentication
```

Рекомендация: rebase для feature веток перед PR — чистая история без лишних merge коммитов.

## Создание Pull Request

```bash
# Через GitHub CLI
gh pr create \
  --title "feat: add user authentication" \
  --body "## Summary
- Implement JWT-based authentication
- Add login/logout endpoints
- Handle token refresh

## Testing
- Unit tests: 45/45 passing
- Manual testing on staging

Closes #123"

# Через GitLab CLI
glab mr create \
  --title "feat: add user authentication" \
  --description "..." \
  --target-branch main
```

## Процесс code review

```bash
# Ревьюер проверяет ветку локально
git fetch origin
git switch feature/user-authentication

# Запустить тесты
npm test

# Оставить комментарии в GitHub/GitLab UI

# После ответа на комментарии — автор обновляет ветку
git commit -am "fix: address review comments"
git push origin feature/user-authentication
```

## Merge стратегии

```bash
# Squash merge (все коммиты ветки → один)
git switch main
git merge --squash feature/user-authentication
git commit -m "feat: add user authentication (#123)"
git branch -d feature/user-authentication

# Merge commit (сохраняет историю ветки)
git merge --no-ff feature/user-authentication

# Rebase и fast-forward (линейная история)
git switch feature/user-authentication
git rebase main
git switch main
git merge feature/user-authentication
```

## Удаление ветки после merge

```bash
# Удалить локальную ветку
git branch -d feature/user-authentication

# Удалить remote ветку
git push origin --delete feature/user-authentication

# GitHub/GitLab может удалять автоматически при merge
# Settings → Delete branch on merge

# Очистить устаревшие remote-tracking ветки
git fetch --prune
```

## Параллельная разработка

```bash
# Несколько разработчиков работают параллельно
Developer 1: feature/user-auth
Developer 2: feature/dark-mode
Developer 3: fix/performance-dashboard

# Каждый независимо:
# - создаёт ветку от main
# - разрабатывает
# - открывает PR
# - получает review
# - merge в main

# Конфликты разрешаются при merge/rebase
```

## Частые ошибки

```bash
# ОШИБКА 1: Слишком большая ветка
# Ветка живёт 2 недели, 50 файлов изменено
# → Разбить на несколько меньших задач

# ОШИБКА 2: Работа напрямую в main
git switch main
git commit -am "quick fix"  # плохо!
# → Всегда создавать ветку, даже для маленьких правок

# ОШИБКА 3: Не обновлять ветку перед PR
# Ветка устарела на 30 коммитов
# → Регулярно синхронизировать с main через rebase

# ОШИБКА 4: Merge без review
git merge feature/new-feature  # без PR
# → Pull Request обязателен для review
```

## Часто задаваемые вопросы

**Как долго должна жить feature ветка?** Как можно меньше — идеально 1-3 дня. Длинные ветки накапливают конфликты. Разбивайте большие задачи на меньшие через feature flags.

**Squash или обычный merge?** Squash создаёт чистую историю в main (один коммит на PR). Обычный merge сохраняет историю разработки. Squash предпочтительнее для main — история чище. Обычный merge — если история разработки важна.

**Можно ли делать merge если тесты не прошли?** Нет. Настройте branch protection чтобы merge блокировался при неудачных тестах: Settings → Branches → Require status checks.

**Как один разработчик работает если нет reviewer?** Можно использовать self-review или автоматическую проверку (линтер, тесты). Для маленьких команд можно установить минимум 1 reviewer.

## Заключение

Feature Branch Workflow — основа совместной разработки: каждая задача в отдельной ветке, merge через Pull Request. Создание: `git switch -c feature/название`. Синхронизация: `git rebase origin/main`. После merge: `git branch -d feature/название`. Это базис для более сложных моделей — [GitHub Flow]({{< relref "github-flow" >}}) и [Gitflow]({{< relref "gitflow" >}}).
