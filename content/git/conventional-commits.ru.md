---
title: "Conventional Commits: стандарт написания сообщений коммитов"
description: "Полное руководство по Conventional Commits. Типы коммитов (feat, fix, docs), scope, breaking changes, автоматизация changelog и semver."
date: 2026-01-03
lastmod: 2026-01-03
draft: false
slug: "conventional-commits"
keywords: ["типы коммитов", "conventional commits", "conventional commits что это", "conventional commits примеры", "conventional commits типы", "формат сообщений коммитов", "commit message правила"]
tags: ["git", "commits", "intermediate"]
categories: ["git"]
---

Conventional Commits — это спецификация для написания структурированных сообщений коммитов. Стандарт позволяет автоматически генерировать changelog, определять тип версионирования (semver) и делать историю проекта читаемой для инструментов и людей.

## Базовая структура

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

Примеры:

```
feat: add user authentication
fix: resolve null pointer in login form
docs: update API endpoint documentation
```

## Типы коммитов

**feat** — новая функциональность:

```
feat: add dark mode toggle
feat(auth): implement OAuth2 login
feat!: redesign user dashboard (breaking change)
```

**fix** — исправление ошибки:

```
fix: prevent XSS in comment input
fix(api): handle empty response from server
fix: correct typo in error message
```

**docs** — изменения в документации:

```
docs: add getting started guide
docs(api): document authentication endpoints
docs: fix broken links in README
```

**style** — форматирование, отступы, отсутствующие точки с запятой (не меняет логику):

```
style: fix indentation in auth module
style: add missing semicolons
style: format according to prettier config
```

**refactor** — рефакторинг без добавления функций и без исправления ошибок:

```
refactor: extract user validation to separate service
refactor(auth): simplify token refresh logic
refactor: rename variables for clarity
```

**perf** — изменения для улучшения производительности:

```
perf: optimize database queries
perf(images): add lazy loading for thumbnails
```

**test** — добавление или исправление тестов:

```
test: add unit tests for auth service
test(api): add integration tests for user endpoints
test: fix flaky timeout test
```

**chore** — обновление зависимостей, конфигурации (без изменения кода):

```
chore: update npm dependencies
chore(deps): bump lodash from 4.17.20 to 4.17.21
chore: add .editorconfig
```

**ci** — изменения в CI/CD конфигурации:

```
ci: add GitHub Actions workflow
ci: fix deployment pipeline
ci(docker): optimize build cache
```

**revert** — откат предыдущего коммита:

```
revert: revert "feat: add dark mode toggle"
This reverts commit a1b2c3d4.
```

## Scope (область изменений)

Scope — опциональное уточнение компонента в скобках:

```
feat(auth): add two-factor authentication
fix(api): handle 404 responses correctly
docs(readme): update installation instructions
test(utils): add tests for date formatter
perf(database): add index to users table
```

Scope помогает быстро понять, какой модуль затронут изменением.

## Breaking Changes

Breaking change — изменение, ломающее обратную совместимость. Обозначается двумя способами:

```bash
# Способ 1: восклицательный знак после типа
feat!: redesign authentication API
fix!: change default port from 8080 to 3000

# Способ 2: BREAKING CHANGE в footer
feat(api): change response format

BREAKING CHANGE: API now returns JSON instead of XML.
Clients must update their parsers.

# Оба способа одновременно (наиболее явно)
feat(auth)!: rename login endpoint

BREAKING CHANGE: /api/login renamed to /api/auth/login
```

Breaking change вызывает мажорное увеличение версии в semver (1.x.x → 2.0.0).

## Тело коммита (Body)

Тело коммита объясняет, ЧТО и ПОЧЕМУ (не как):

```
fix: prevent race condition in session handling

Previously, concurrent requests could create duplicate sessions
when users logged in simultaneously. This was causing data
inconsistency in the user table.

The fix adds a database-level unique constraint and
handles the race condition with a retry mechanism.
```

Отделяется от заголовка пустой строкой. Каждая строка — не длиннее 72 символов.

## Footer (сноски)

Footer содержит дополнительные метаданные:

```
feat: add user profile page

Implement profile editing and avatar upload.

Closes #123
Fixes #456
Reviewed-by: Alice <alice@example.com>
Co-authored-by: Bob <bob@example.com>
```

Распространённые footer-токены: `Closes`, `Fixes`, `Refs`, `BREAKING CHANGE`, `Co-authored-by`.

## Примеры реальных коммитов

```
feat(auth): add JWT token refresh

Implement automatic token refresh when token expires.
Refresh happens silently without user logout.

Closes #87

---

fix(validation): validate email format on registration

Previously empty email was accepted during signup.
Added RFC 5322 compliant email validation.

Fixes #234

---

feat(api)!: change pagination format

BREAKING CHANGE: pagination now uses cursor-based
instead of offset-based approach.
Update all API clients to use `cursor` parameter
instead of `page` and `offset`.

---

chore(deps): update dependencies

- lodash: 4.17.20 → 4.17.21 (security fix)
- axios: 0.24.0 → 0.27.2
- typescript: 4.5.4 → 4.7.4
```

## Автоматизация на основе Conventional Commits

**Semantic Release** — автоматический релиз и changelog:

```bash
# Установить
npm install -D semantic-release

# Определяет версию по типам коммитов:
# feat:         1.0.0 → 1.1.0 (minor)
# fix:          1.0.0 → 1.0.1 (patch)
# BREAKING:     1.0.0 → 2.0.0 (major)
```

**Commitlint** — валидация сообщений коммитов:

```bash
# Установить
npm install -D @commitlint/cli @commitlint/config-conventional

# commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional']
}

# Проверить последний коммит
npx commitlint --from HEAD~1 --to HEAD --verbose
```

**Conventional Changelog** — генерация CHANGELOG.md:

```bash
# Установить
npm install -D conventional-changelog-cli

# Сгенерировать changelog
npx conventional-changelog -p conventional -i CHANGELOG.md -s
```

## Настройка git-cz (commitizen)

Интерактивный помощник для написания Conventional Commits:

```bash
# Установить
npm install -g commitizen cz-conventional-changelog

# Настроить
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc

# Использование — вместо git commit:
git cz
# Запустит интерактивный мастер:
# ? Select the type of change: (feat/fix/docs/...)
# ? What is the scope? (optional)
# ? Write a short description:
# ...
```

## Практические рекомендации

```
✓ Используйте повелительное наклонение: "add" не "added"
✓ Первая буква строчная в description
✓ Нет точки в конце description
✓ Заголовок до 72 символов
✓ Отделяйте тело от заголовка пустой строкой
✓ Будьте конкретны: "fix null pointer in login" не "fix bug"
✓ Один коммит — одно изменение
✗ Не используйте: "wip", "update", "changes", "misc"
```

## Часто задаваемые вопросы

**Обязательно ли следовать Conventional Commits?** Нет, но это стандарт де-факто в open source. Помогает автоматизировать changelog и версионирование, делает историю более читаемой.

**Как выбрать между `fix` и `refactor`?** `fix` — исправление ошибочного поведения. `refactor` — изменение кода без изменения поведения (оптимизация, упрощение). Если баг был исправлен попутно — это `fix`.

**Что делать если коммит затрагивает несколько типов?** Разбейте на несколько коммитов. Если невозможно — используйте наиболее значимый тип (feat > fix > refactor > chore).

**Нужен ли scope?** Необязательно. Полезен в больших проектах с несколькими модулями. В маленьких — излишен.

**Как Semantic Release определяет версию?** feat → minor bump (1.0.0 → 1.1.0), fix/perf → patch bump (1.0.0 → 1.0.1), BREAKING CHANGE → major bump (1.0.0 → 2.0.0). docs/chore/style/test — не влияют на версию.

## Заключение

Conventional Commits — это соглашение, которое превращает сообщения коммитов в структурированные данные. Основа: `type(scope): description`. Типы: feat, fix, docs, style, refactor, perf, test, chore, ci. Breaking changes обозначаются `!` или BREAKING CHANGE в footer. Позволяет автоматизировать changelog через Semantic Release. Подробнее о лучших практиках написания коммитов — [лучшие практики коммитов]({{< relref "luchshie-praktiki-kommitov" >}}).
