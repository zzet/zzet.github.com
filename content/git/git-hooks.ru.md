---
title: "Git хуки (hooks): автоматизация проверок при коммитах"
description: "Полное руководство по Git хукам. pre-commit, commit-msg, pre-push хуки. Создание хуков вручную, Husky для Node.js, pre-commit framework."
date: 2026-01-14
lastmod: 2026-01-14
draft: false
slug: "git-hooks"
keywords: ["git hooks", "git хуки", "pre-commit hook git", "pre-commit хук настройка", "как создать git хук", "git hooks примеры использования", "commit-msg hook"]
tags: ["git", "intermediate", "automation"]
categories: ["git"]
---

Git хуки — это скрипты, которые автоматически запускаются при определённых событиях: коммите, push, merge. Используются для запуска тестов, проверки форматирования кода, валидации сообщений коммитов.

## Где находятся хуки

Хуки хранятся в `.git/hooks/`:

```bash
ls .git/hooks/
# applypatch-msg.sample
# commit-msg.sample
# post-update.sample
# pre-applypatch.sample
# pre-commit.sample      ← пример pre-commit
# pre-push.sample
# pre-rebase.sample
# prepare-commit-msg.sample
# update.sample
```

Файлы `.sample` — примеры. Чтобы активировать хук — убрать `.sample` из имени и дать права на исполнение.

## Основные клиентские хуки

**pre-commit** — запускается перед созданием коммита:

```bash
# .git/hooks/pre-commit
#!/bin/sh
# Запустить линтер
npm run lint

# Если линтер упал — прервать коммит
if [ $? -ne 0 ]; then
    echo "Lint failed. Fix errors before committing."
    exit 1
fi

exit 0
```

**commit-msg** — проверяет сообщение коммита:

```bash
# .git/hooks/commit-msg
#!/bin/sh
# Проверить что сообщение соответствует Conventional Commits
COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Паттерн: type(scope): description
if ! echo "$COMMIT_MSG" | grep -qE "^(feat|fix|docs|style|refactor|perf|test|chore|ci)(\(.+\))?: .+"; then
    echo "ERROR: Commit message must follow Conventional Commits format"
    echo "Example: feat: add user authentication"
    exit 1
fi
```

**pre-push** — запускается перед push:

```bash
# .git/hooks/pre-push
#!/bin/sh
# Запустить тесты перед push
npm test

if [ $? -ne 0 ]; then
    echo "Tests failed. Fix tests before pushing."
    exit 1
fi
```

## Создание хука вручную

```bash
# Создать pre-commit хук
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
echo "Running pre-commit checks..."

# Проверить форматирование
npx prettier --check "src/**/*.{js,ts,jsx,tsx}"
if [ $? -ne 0 ]; then
    echo "Code is not formatted. Run: npx prettier --write ."
    exit 1
fi

echo "Pre-commit checks passed."
exit 0
EOF

# Дать права на выполнение (обязательно!)
chmod +x .git/hooks/pre-commit

# Проверить
git commit --allow-empty -m "test"
```

## Серверные хуки

Серверные хуки запускаются на Git сервере:

```bash
# pre-receive — перед получением push
# Расположение: на сервере в hooks/pre-receive

#!/bin/sh
# Запретить force push в main
while read oldrev newrev refname; do
    if [ "$refname" = "refs/heads/main" ]; then
        if [ "$oldrev" != "0000000000000000000000000000000000000000" ]; then
            if ! git merge-base --is-ancestor "$oldrev" "$newrev"; then
                echo "ERROR: Force push to main is not allowed."
                exit 1
            fi
        fi
    fi
done
```

## Husky — хуки для Node.js проектов

Husky — инструмент для управления Git хуками через package.json:

```bash
# Установить Husky
npm install -D husky

# Активировать
npx husky init

# Это создаёт .husky/ папку и добавляет prepare скрипт

# Добавить pre-commit хук
echo "npx lint-staged" > .husky/pre-commit

# Установить lint-staged
npm install -D lint-staged

# Конфигурация в package.json
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts,jsx,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{css,scss}": [
      "prettier --write"
    ]
  },
  "scripts": {
    "prepare": "husky"
  }
}
```

```bash
# Добавить commit-msg хук с commitlint
npm install -D @commitlint/cli @commitlint/config-conventional

echo "npx --no commitlint --edit \$1" > .husky/commit-msg

# Создать commitlint.config.js
echo "module.exports = { extends: ['@commitlint/config-conventional'] }" > commitlint.config.js
```

## pre-commit framework (Python проекты)

Для Python и многоязычных проектов:

```bash
# Установить pre-commit
pip install pre-commit

# Создать конфигурацию .pre-commit-config.yaml
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-merge-conflict

  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
EOF

# Установить хуки
pre-commit install

# Запустить вручную на всех файлах
pre-commit run --all-files
```

## Пропуск хуков

```bash
# Пропустить pre-commit хук
git commit --no-verify -m "WIP: work in progress"
# или
git commit -n -m "Emergency fix"

# Пропустить pre-push хук
git push --no-verify

# Когда это оправдано:
# - Срочные hotfix в нерабочее время
# - WIP коммиты для резервного копирования
# - Временный обход при отладке CI
```

## Общий доступ к хукам в команде

Хуки в `.git/hooks/` не коммитятся. Для общего использования:

```bash
# Способ 1: хранить в папке scripts/hooks
mkdir -p scripts/hooks
cp .git/hooks/pre-commit scripts/hooks/

# Инструкция для команды:
# git config core.hooksPath scripts/hooks

# Способ 2: Husky (для Node.js) — .husky/ папка коммитится
# Способ 3: pre-commit framework — .pre-commit-config.yaml коммитится
```

## Практические примеры хуков

```bash
# pre-commit: запуск тестов на изменённых файлах
#!/bin/sh
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep ".js$")
if [ -n "$STAGED_FILES" ]; then
    npm test -- --testPathPattern="$(echo $STAGED_FILES | tr ' ' '|')"
fi

# commit-msg: добавить ID задачи из имени ветки
#!/bin/sh
BRANCH=$(git rev-parse --abbrev-ref HEAD)
TICKET=$(echo $BRANCH | grep -oE "PROJ-[0-9]+")
if [ -n "$TICKET" ]; then
    sed -i "1s/^/$TICKET: /" $1
fi
```

## Часто задаваемые вопросы

**Почему хук не запускается?** Чаще всего — нет прав на выполнение. Выполните `chmod +x .git/hooks/pre-commit`.

**Как отключить хук для одного коммита?** `git commit --no-verify` или `git commit -n`. Не злоупотребляйте — хуки защищают качество кода.

**Как поделиться хуками с командой?** Хуки в `.git/hooks/` не попадают в репозиторий. Используйте Husky (Node.js), pre-commit framework (Python/универсальный) или `git config core.hooksPath <dir>` с коммитом папки хуков.

**Можно ли запускать хуки асинхронно?** Только специфические (post-commit, post-receive). Большинство хуков синхронные — блокируют операцию до завершения.

## Заключение

Git хуки автоматизируют проверки качества кода. `pre-commit` — форматирование и линтинг перед коммитом. `commit-msg` — валидация сообщения. `pre-push` — тесты перед push. Для Node.js — Husky + lint-staged. Для Python/универсальных проектов — pre-commit framework. При срочных случаях можно пропустить с `--no-verify`. Подробнее о стандарте сообщений коммитов — [Conventional Commits]({{< relref "conventional-commits" >}}).
