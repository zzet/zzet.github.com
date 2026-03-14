---
title: "Сообщение merge коммита в Git: как изменить стандартное и задать своё"
description: "Как задать своё сообщение при git merge: флаг -m, --edit. Изменить созданное сообщение через amend. Squash merge для чистого сообщения."
date: 2026-03-14
lastmod: 2026-03-14
draft: false
slug: "git-merge-commit-message"
keywords: ["git merge message", "сообщение merge коммита git", "изменить сообщение merge коммита", "git merge commit text"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Когда вы объединяете две ветки в Git, автоматически создаётся так называемый merge коммит — специальный коммит, который объединяет историю двух веток. По умолчанию Git генерирует стандартное сообщение для такого коммита:

```
Merge branch 'feature/auth' into main
```

Это сообщение содержит базовую информацию, но часто недостаточно информативно. Вы можете захотеть добавить дополнительные детали, ссылку на issue, описание функции или другую полезную информацию. Вот здесь и пригодится возможность контролировать сообщение merge коммита.

В этой статье мы разберёмся, как изменить сообщение merge коммита различными способами: задать своё сообщение во время merge, отредактировать уже созданное сообщение, использовать squash merge для чистой истории, и автоматизировать процесс через git hooks.

## Стандартное сообщение merge коммита

Когда вы выполняете простую команду `git merge`, Git создаёт коммит с автоматическим сообщением:

```bash
git merge feature/authentication
# Создаёт коммит с сообщением:
# "Merge branch 'feature/authentication' into main"
```

Это сообщение совершенно валидно для простых случаев, особенно когда ветка маленькая и одна особая. Однако для больших merge операций, которые объединяют значительные изменения, стоит добавить больше контекста.

Посмотрите историю коммитов:

```bash
git log --oneline

# Пример вывода:
# a1b2c3d Merge branch 'feature/authentication' into main
# x9y8z7w Add password validation
# m5n4o3p Add login form
# e2f1g0h Update documentation
```

Как видите, merge коммит выглядит довольно скромно. Если потом вы захотите узнать, что было объединено, вам пришлось бы смотреть на детали коммита отдельно.

## Задать своё сообщение при merge

### Способ 1: Флаг -m (самый простой)

Самый прямой способ — указать сообщение прямо в команде merge с флагом `-m`:

```bash
git merge feature/authentication -m "feat: add user authentication system

- Implement login form with validation
- Add password hashing and security
- Create session management
- Write integration tests

Closes #42
Reviewed-by: John Doe"
```

Результат:

```
Merge branch 'feature/authentication' into main

feat: add user authentication system

- Implement login form with validation
- Add password hashing and security
- Create session management
- Write integration tests

Closes #42
Reviewed-by: John Doe
```

Когда вы используете флаг `-m`, весь текст становится сообщением merge коммита. Первая строка (до первой пустой строки) считается заголовком, остальное — описанием.

### Способ 2: Флаг --edit (редактирование в редакторе)

Если вы хотите отредактировать сообщение в текстовом редакторе (обычно vim или nano), используйте флаг `--edit`:

```bash
git merge feature/authentication --edit
```

Git откроет ваш редактор по умолчанию с предзаполненным стандартным сообщением:

```
Merge branch 'feature/authentication' into main
#
# It looks like you may be committing a merge.
# If this is not correct, please remove the line above.
```

Вы можете отредактировать эту заголовок и добавить описание:

```
Merge branch 'feature/authentication' into main

feat: complete authentication implementation
- Login/logout functionality
- Password recovery
- Two-factor authentication support

Closes #42
```

Сохраните файл и закройте редактор (`:wq` в vim), и merge будет завершён с вашим сообщением.

### Способ 3: Комбинация флагов

Вы можете комбинировать флаги для более гибкой работы:

```bash
# Merge без создания коммита (подготовка вручную)
git merge --no-commit feature/authentication

# Потом создать коммит с вашим сообщением
git commit -m "feat: add authentication

Complete implementation of user authentication including:
- Registration and login flow
- Email verification
- Password reset functionality

Closes #42"
```

Это полезно, если вы хотите сначала посмотреть изменения перед тем, как зафиксировать сообщение.

## Отключить редактирование сообщения (--no-edit)

Иногда вы просто хотите merge без каких-либо вопросов, используя стандартное сообщение:

```bash
git merge feature/authentication --no-edit

# Аналогично:
git merge feature/authentication -m "Merge branch 'feature/authentication' into main"
```

Флаг `--no-edit` полезен при автоматизации скриптов, когда вы не хотите интерактивного вмешательства.

## Изменить уже созданное сообщение merge коммита

Если вы уже создали merge коммит со стандартным или неправильным сообщением, вы можете изменить его с помощью `git commit --amend`:

### Редактирование в текстовом редакторе

```bash
# Отредактировать последний коммит (merge)
git commit --amend

# Это откроет редактор с текущим сообщением merge коммита
```

Отредактируйте сообщение по своему вкусу и сохраните.

### Замена всего сообщения

```bash
# Заменить сообщение на новое
git commit --amend -m "feat: add authentication system

Implements complete user authentication:
- Login/logout with session management
- Password hashing and security
- Email verification
- Password recovery flow

Tests: 45 new tests added
Closes #42"
```

### Важно о --amend с push

Если вы уже отправили merge коммит на сервер (`git push`), изменение его с помощью `--amend` потребует force push:

```bash
# Изменить сообщение
git commit --amend -m "новое сообщение"

# Force push (осторожно, если работаете в команде!)
git push --force-with-lease

# или для главной ветки (рекомендуется)
git push --force-with-lease origin main
```

Флаг `--force-with-lease` безопаснее, чем `--force`, так как отменит push только если никто другой не сделал push в эту ветку.

## Настройка шаблона сообщения

Если у вас есть стандартный формат для всех сообщений (например, в организации), вы можете создать шаблон:

### Создание файла шаблона

Создайте файл `~/.gitmessage`:

```
# Вставляйте сообщение выше этой строки
# Формат:
# <type>: <subject>
#
# <body>
#
# <footer>
#
# Тип: feat, fix, docs, style, refactor, perf, test, chore, ci
# Тема (subject): 50 символов максимум
# Тело (body): разъясняет детали, обёртывание на 72 символа
# Нижний колонтитул (footer): закрыть issues (Closes #123)
```

### Использование шаблона в конфигурации

```bash
# Настроить Git использовать этот шаблон
git config --global commit.template ~/.gitmessage

# Или для конкретного репозитория
git config commit.template ~/.gitmessage
```

Теперь при каждом редактировании коммита (включая merge с флагом `--edit`) будет использоваться этот шаблон.

## Merge с squash (одно сообщение для всех коммитов)

Иногда вместо обычного merge лучше использовать squash merge. Это создаёт один коммит со всеми изменениями из ветки, вместо создания merge коммита с полной историей.

```bash
# Выполнить squash merge
git merge --squash feature/authentication

# Git подготовит все изменения, но не создаёт коммит
# Вам нужно создать коммит самостоятельно с вашим сообщением
git commit -m "feat: add authentication system

Complete implementation includes:
- User registration and login
- Email verification
- Password recovery
- Two-factor authentication

This squash merges the entire feature/authentication branch into main.
Closes #42"
```

Результат squash merge:

```bash
git log --oneline

# Вывод:
# a1b2c3d feat: add authentication system
# m5n4o3p Update documentation
# ... (все коммиты из feature ветки исчезли, стали одним)
```

### Когда использовать squash merge

- **Маленькие функции** с множеством промежуточных коммитов
- **Когда нужна чистая история** в main ветке
- **Для работы с фишами** от разработчиков без определённого процесса коммитов
- **В GitHub**, при использовании опции "Squash and merge"

Однако squash merge **теряет историю** отдельных коммитов, что может быть невыгодно для больших фич.

## Merge в GitHub/GitLab/Bitbucket

Когда вы используете веб-интерфейс GitHub для слияния Pull Request, вы видите три опции:

### 1. Create a merge commit

```bash
# Эквивалент:
git merge feature/branch --no-edit

# Создаёт merge коммит с сообщением:
# "Merge pull request #42 from username/feature/branch"
```

### 2. Squash and merge

```bash
# Эквивалент:
git merge --squash feature/branch
git commit -m "Title of PR (#42)"

# Все коммиты из PR объединяются в один
```

### 3. Rebase and merge

```bash
# Эквивалент:
git rebase main feature/branch
git merge --ff-only feature/branch

# Никакого merge коммита не создаётся, история остаётся чистой
```

В GitHub можно также отредактировать сообщение merge коммита перед нажатием кнопки merge.

## Автоматизация через prepare-commit-msg хук

Если вы хотите автоматически добавлять информацию в сообщение merge коммита (например, номер issue из названия ветки), используйте git hook:

Создайте файл `.git/hooks/prepare-commit-msg`:

```bash
#!/bin/bash

# Получить тип коммита (merge, squash и т.д.)
COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2

# Если это merge коммит
if [ "$COMMIT_SOURCE" = "merge" ]; then
  # Получить текущую ветку
  BRANCH_NAME=$(git rev-parse --abbrev-ref HEAD)

  # Извлечь номер issue из названия ветки (например, feature/auth-123)
  ISSUE_NUM=$(echo $BRANCH_NAME | grep -oE '[0-9]+' | head -1)

  if [ -n "$ISSUE_NUM" ]; then
    # Добавить ссылку на issue в конец сообщения
    echo "" >> $COMMIT_MSG_FILE
    echo "Closes #$ISSUE_NUM" >> $COMMIT_MSG_FILE
  fi
fi
```

Сделайте файл исполняемым:

```bash
chmod +x .git/hooks/prepare-commit-msg
```

Теперь при merge с флагом `--edit` номер issue будет автоматически добавлен в сообщение.

## Conventional Commits для merge сообщений

Если вы следуете {{< relref "conventional-commits" >}}, вот как должны выглядеть merge сообщения:

```bash
git merge feature/authentication -m "feat: add user authentication

BREAKING CHANGE: Password hashing algorithm changed
Users will need to reset passwords after update

Implementation details:
- bcrypt for password hashing
- JWT for session tokens
- Email verification required

Closes #42
Reviewed-by: Jane Smith"
```

Формат:
- **Тип** (feat, fix, docs, style, refactor, perf, test, chore, ci)
- **Область** (опционально) — например, auth, api, ui
- **Описание** — что было объединено
- **Тело** — дополнительные детали
- **Нижний колонтитул** — ссылки и мета-информация

## Практические примеры

### Пример 1: Обычный merge с информативным сообщением

```bash
git merge feature/payment-integration -m "feat: integrate stripe payment

Add complete payment processing:
- Stripe API integration
- Payment status tracking
- Email receipt generation
- Webhook handling for payment events

Tests: 52 test cases added, 100% coverage
Performance: Optimised for <100ms response time
Security: PCI compliance verified

Closes #234
Co-authored-by: Alice <alice@example.com>"
```

### Пример 2: Hotfix с срочным merge

```bash
git merge hotfix/security-patch -m "fix: critical security vulnerability

CRITICAL: SQL injection vulnerability in user search
All deployments should be updated immediately

This patch:
- Fixes parameterized queries
- Updates all user inputs sanitization
- Adds input validation tests

Affects: Production
Fixed-by: John Smith
Reviewed-by: Security Team"
```

### Пример 3: Release merge

```bash
git merge release/v2.0.0 -m "chore: release version 2.0.0

Release 2.0.0 includes:
- 156 commits
- 23 new features
- 45 bug fixes
- 12 performance improvements

Changelog: https://github.com/project/releases/tag/v2.0.0
Release notes: docs/RELEASE_NOTES.md"
```

## FAQ

**Вопрос: Как автоматически добавлять номер issue в merge сообщение из названия PR?**
GitHub и GitLab частично автоматизируют это через веб-интерфейс. Вы можете также использовать git hooks (prepare-commit-msg) для автоматизации локально, как описано выше.

**Вопрос: Какие соглашения по сообщениям merge рекомендуются?**
Используйте {{< relref "conventional-commits" >}} с типами (feat, fix, etc.). Всегда добавляйте информацию о чём был merge и почему. Используйте "Closes #123" для привязки к issues.

**Вопрос: Когда использовать squash merge вместо обычного merge?**
Для маленьких features с множеством промежуточных коммитов используйте squash. Для больших features или долгоживущих веток сохраняйте историю обычным merge.

**Вопрос: Как изменить сообщение merge, если я уже его отправил (push)?**
Используйте `git commit --amend` для изменения, затем `git push --force-with-lease` для отправки. Будьте осторожны в совместной разработке — согласуйте это с командой.

**Вопрос: GitHub предлагает использовать заголовок PR как merge сообщение, это хорошо?**
Да, если заголовок PR информативен и следует соглашениям. Вы можете отредактировать сообщение прямо перед merge в GitHub, добавив описание и ссылки на issues.

## Заключение

Сообщение merge коммита — это важная часть истории проекта. Хорошо написанное merge сообщение:

1. **Помогает разработчикам понимать** что и почему было объединено
2. **Связывает код с issues** через "Closes #123"
3. **Документирует процесс разработки** для будущего анализа
4. **Облегчает поиск** в истории проекта

Основные способы задать сообщение:
- **`git merge branch -m "message"`** — простой и прямой
- **`git merge branch --edit`** — для редактирования в редакторе
- **`git merge --squash`** — для чистой истории маленьких фич
- **`git commit --amend`** — для изменения уже созданного merge коммита

Рекомендуем изучить {{< relref "git-merge" >}}, {{< relref "conventional-commits" >}} и {{< relref "git-commit-amend" >}} для глубокого понимания работы с коммитами и merge операциями. Также будет полезно прочитать {{< relref "git-revert-merge-commit" >}} для понимания как отменять merge, если что-то пошло не так.
