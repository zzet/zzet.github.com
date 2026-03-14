---
title: "Соглашение об именовании веток в Git: лучшие практики"
description: "Полное руководство по именованию веток в Git. Схемы, типы веток, префиксы, Git Flow, GitHub Flow, примеры и автоматизация."
date: 2026-02-17
lastmod: 2026-02-17
draft: false
slug: "nazvanie-vetok-git"
keywords: ["как называть ветки в git", "соглашение об именовании веток", "naming convention git", "git branch naming", "имена веток git примеры", "как назвать ветку git"]
tags: ["git", "branch", "best-practices"]
categories: ["git"]
---

Хаотичные названия веток превращают репозиторий в беспорядок: непонятно, что делает каждая ветка, кто её создал, к какой задаче она относится. Хорошее соглашение об именовании решает все эти проблемы и занимает пять минут на настройку.

## Зачем нужны соглашения об именовании

Без стандарта история веток выглядит так:

```
fix
test123
Johns_branch
new-feature
feature2
TEMP
```

Со стандартом:

```
feature/123-user-authentication
bugfix/456-fix-cart-total
hotfix/789-critical-security-patch
docs/update-readme
release/v2.1.0
```

Сразу видно: тип изменения, номер задачи, краткое описание. Команда может ориентироваться без чтения кода.

## Основные компоненты названия ветки

Хорошее название ветки состоит из трёх частей:

```
<тип>/<номер-задачи>-<описание>
```

Например: `feature/PROJ-123-implement-oauth2`

**Тип** — категория изменения: `feature`, `bugfix`, `hotfix`, `docs`, `refactor`, `test`, `release`.

**Номер задачи** — идентификатор из трекера задач (Jira, Linear, GitHub Issues). Позволяет связать ветку с задачей: `PROJ-123`, `#456`, `GH-789`.

**Описание** — краткое описание через дефис в нижнем регистре: `user-authentication`, `fix-cart-total`.

## Популярные схемы именования

**Простая (для небольших проектов):**

```bash
git switch -c feature-user-auth
git switch -c bugfix-login-error
git switch -c docs-readme-update
```

**С типом и слешем (рекомендуется):**

```bash
git switch -c feature/user-authentication
git switch -c bugfix/login-form-validation
git switch -c docs/setup-instructions
git switch -c hotfix/critical-security-patch
```

**С номером задачи (для командных проектов):**

```bash
git switch -c feature/123-user-authentication
git switch -c bugfix/456-fix-login-error
git switch -c feature/PROJ-789-shopping-cart
```

## Типы веток и их назначение

**feature/** — новая функциональность:

```bash
git switch -c feature/user-profile
git switch -c feature/payment-integration
```

**bugfix/** — исправление ошибок в разработке:

```bash
git switch -c bugfix/cart-total-calculation
git switch -c bugfix/login-redirect-loop
```

**hotfix/** — срочное исправление в production (обычно ветвится от main):

```bash
git switch -c hotfix/security-vulnerability
git switch -c hotfix/payment-gateway-error
```

**release/** — подготовка к релизу:

```bash
git switch -c release/v1.2.0
git switch -c release/2026-q1
```

**docs/** — обновление документации:

```bash
git switch -c docs/api-documentation
git switch -c docs/deployment-guide
```

**refactor/** — рефакторинг без изменения функционала:

```bash
git switch -c refactor/simplify-auth-logic
```

**test/** — добавление тестов:

```bash
git switch -c test/integration-tests-auth
```

**chore/** — технические задачи (обновление зависимостей и т.д.):

```bash
git switch -c chore/update-dependencies
git switch -c chore/migrate-to-typescript
```

## Разделители: дефис vs нижнее подчёркивание

Рекомендуется использовать **дефис** (`-`) как разделитель слов в описании ветки. Нижнее подчёркивание тоже работает, но менее читабельно:

```bash
# Хорошо:
feature/user-authentication
feature/shopping-cart

# Менее предпочтительно:
feature/user_authentication
feature/shopping_cart
```

Слеш (`/`) используется для разделения типа и описания. Git обрабатывает его особым образом — в некоторых клиентах создаёт «папки» для организации веток.

## Git Flow vs GitHub Flow: разные стандарты

**Git Flow** (сложная модель, для крупных проектов):

```
main          ← стабильный код
develop       ← ветка разработки
feature/...   ← новые функции, ветвятся от develop
release/...   ← подготовка релиза, ветвится от develop
hotfix/...    ← срочные исправления, ветвится от main
```

**GitHub Flow** (простая модель, для большинства проектов):

```
main          ← основная ветка
feature/...   ← всё остальное ветвится от main и сливается в main
```

Выбор модели определяет схему именования. GitHub Flow проще и подходит большинству команд.

## Правила хорошего именования

**Используйте нижний регистр:** `feature/user-auth`, а не `Feature/User-Auth`.

**Не используйте пробелы:** `feature/user-auth`, а не `feature/user auth`.

**Ограничивайте длину:** название ветки не должно быть слишком длинным. `feature/123-add-oauth2` лучше, чем `feature/123-add-google-oauth2-authentication-with-refresh-tokens`.

**Будьте описательными:** `bugfix/456-fix-negative-cart-total` лучше, чем `bugfix/456-fix`.

**Избегайте специальных символов:** только буквы, цифры, дефисы, слеши, нижние подчёркивания.

## Практические примеры

```bash
# Создание ветки по соглашению
git switch -c feature/PROJ-123-implement-oauth2
git switch -c bugfix/PROJ-456-fix-pagination
git switch -c docs/PROJ-789-api-documentation

# Просмотр веток по типу
git branch -a | grep feature/
git branch -a | grep bugfix/

# Фильтрация в git log
git log --oneline --all | grep feature/

# Создание ветки с датой (для временных веток)
git switch -c wip/$(date +%Y%m%d)-experiment-new-algo

# Удалить все слитые feature-ветки
git branch --merged main | grep feature/ | xargs git branch -d
```

## Автоматизация проверки соглашений

В pre-push хуке (`.git/hooks/pre-push`) или через Husky можно проверять название ветки:

```bash
#!/bin/bash
branch=$(git rev-parse --abbrev-ref HEAD)
valid_pattern="^(main|develop|feature|bugfix|hotfix|release|docs|refactor|test|chore)/[a-z0-9/-]+$"

if ! [[ $branch =~ $valid_pattern ]]; then
  echo "ERROR: Branch name '$branch' doesn't match convention"
  echo "Expected format: <type>/<description>"
  echo "Types: feature, bugfix, hotfix, release, docs, refactor, test, chore"
  exit 1
fi
```

## Часто задаваемые вопросы

**Какую схему именования выбрать для команды?** Для небольших команд — GitHub Flow с `feature/`, `bugfix/`, `hotfix/`. Для крупных проектов с релизным циклом — Git Flow. Главное — договориться и задокументировать.

**Нужно ли добавлять номер задачи в название ветки?** Если есть трекер задач (Jira, GitHub Issues, Linear) — да. Это позволяет быстро перейти к задаче из ветки и наоборот. Автоматические интеграции часто используют этот номер.

**Может ли название ветки содержать спецсимволы?** Технически Git допускает многое, но рекомендуется ограничиться: буквы (a-z, A-Z), цифры (0-9), дефис (-), нижнее подчёркивание (_), слеш (/), точка (.). Пробелы, @, ~ и другие специальные символы создают проблемы.

**Как автоматизировать проверку соглашений?** Через Git hooks (pre-commit, pre-push) или CI/CD. Также можно использовать Husky с commitlint для единого набора правил.

## Заключение

Хорошее соглашение об именовании веток — это договорённость команды. Выберите схему (`type/description` или `type/id-description`), задокументируйте её в README или CONTRIBUTING.md, и придерживайтесь. Автоматизируйте проверку через hooks.

Подробнее о создании веток — [как создать ветку в Git]({{< relref "kak-sozdat-vetku-git" >}}). Об общих принципах работы с ветками — [ветки в Git]({{< relref "vetki-v-git" >}}).
