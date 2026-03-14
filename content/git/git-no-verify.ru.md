---
title: "git commit --no-verify: пропуск Git хуков"
description: "Флаг --no-verify в Git: пропуск pre-commit и commit-msg хуков. Когда использовать, риски, лучшие практики. Альтернативы --no-verify."
date: 2026-01-20
lastmod: 2026-01-20
draft: false
slug: "git-no-verify"
keywords: ["git no verify", "git commit --no-verify", "пропустить hook git", "обойти pre-commit hook", "git push --no-verify", "skip git hooks"]
tags: ["git", "intermediate"]
categories: ["git"]
---

`--no-verify` (или `-n`) — флаг для `git commit` и `git push`, который отключает выполнение Git хуков. Полезен в экстренных ситуациях, но злоупотребление им разрушает систему автоматических проверок.

## Что делает --no-verify

```bash
# Обычный коммит — запускает pre-commit и commit-msg хуки
git commit -m "feat: add feature"
# Running pre-commit hook...
# Running commit-msg hook...

# С --no-verify — хуки пропускаются
git commit --no-verify -m "feat: add feature"
# (хуки не запускаются)

# Сокращение
git commit -n -m "feat: add feature"
```

Флаг пропускает:
- `pre-commit` — проверки кода, форматирование, тесты
- `commit-msg` — валидация сообщения коммита

## git push --no-verify

```bash
# Обычный push — запускает pre-push хук
git push origin main
# Running pre-push hook...
# Running tests...

# С --no-verify — pre-push пропускается
git push --no-verify origin main
```

## Когда использовать --no-verify

Допустимые сценарии:

```bash
# 1. Срочный hotfix в нерабочее время
git commit --no-verify -m "fix: critical security patch"
# Позже исправить что не прошло хуки

# 2. WIP коммит для резервного копирования
git commit --no-verify -m "WIP: save progress before meeting"
# Будет squashed перед merge

# 3. Коммит при заведомо известном нарушении
# (например, временно отключён тест)
git commit --no-verify -m "chore: disable failing test temporarily"

# 4. Отладка хука
# Когда сам хук сломан и нужно зафиксировать исправление
git commit --no-verify -m "fix: repair pre-commit hook"
```

## Когда НЕ использовать --no-verify

```
✗ Чтобы обойти линтер который «мешает»
✗ Потому что лень исправить предупреждения
✗ При каждом коммите по умолчанию
✗ Чтобы закоммитить заведомо плохой код в main/master
✗ При работе с общими ветками (это нарушает соглашение команды)
```

## Риски использования --no-verify

```
Немедленные риски:
- Код с ошибками линтера попадает в репозиторий
- Нестандартное сообщение коммита нарушает changelog
- Непрошедшие тесты становятся частью истории

Долгосрочные риски:
- Команда теряет доверие к автоматическим проверкам
- Технический долг накапливается незаметно
- Сложнее отслеживать что было проверено
```

## Альтернативы --no-verify

Вместо пропуска хуков — решить конкретную проблему:

```bash
# Временно отключить проверку одного правила eslint
// eslint-disable-next-line no-console
console.log(debug)

# Добавить файл в исключения prettier
# .prettierignore
legacy-file.js

# Пропустить конкретный хук через переменную
# В хуке:
if [ "$SKIP_LINT" = "1" ]; then
    exit 0
fi
npm run lint

# Использование:
SKIP_LINT=1 git commit -m "WIP: skip lint for now"

# Использовать git stash для частичных изменений
git stash
git commit -m "feat: stable part"
git stash pop
```

## Обнаружение злоупотребления --no-verify

В командах можно настроить серверный хук для отслеживания:

```bash
# pre-receive хук на сервере (не обходится --no-verify)
#!/bin/sh
while read oldrev newrev refname; do
    # Проверить каждый новый коммит
    git log $oldrev..$newrev --format="%H %s" | while read hash subject; do
        # Валидация сообщения коммита на сервере
        if ! echo "$subject" | grep -qE "^(feat|fix|docs|style|refactor|perf|test|chore)"; then
            echo "ERROR: Invalid commit message: $subject"
            echo "Must follow Conventional Commits format"
            exit 1
        fi
    done
done
```

Серверные хуки (`pre-receive`, `update`) выполняются на сервере и не могут быть обойдены клиентским `--no-verify`.

## Логирование использования --no-verify

```bash
# Отслеживать использование (pre-commit хук)
#!/bin/sh
# Записать в лог что был пропущен хук
# (этот код никогда не запустится при --no-verify, но
# можно добавить проверку через git log)

# Проверить историю на коммиты без хуков (косвенно)
# — сложно, --no-verify не оставляет следов в истории
```

## Часто задаваемые вопросы

**Можно ли обойти серверные хуки через --no-verify?** Нет. `--no-verify` отключает только клиентские хуки (`pre-commit`, `commit-msg`, `pre-push`). Серверные хуки (`pre-receive`, `update`, `post-receive`) выполняются на сервере и не зависят от клиентских флагов.

**Как настроить хуки так чтобы их нельзя было обойти?** Только через серверные хуки. Или через CI/CD — пайплайн проверяет код при каждом push независимо от хуков.

**Остаётся ли след использования --no-verify?** В git log нет явного указания. Косвенно можно заметить по нарушениям в сообщениях коммитов или проблемам в коде. Серверный `pre-receive` хук может добавить проверку на стороне сервера.

**Что делать если хук сломался и блокирует работу?** Временно используйте `--no-verify`, но немедленно исправьте хук. Не игнорируйте — сломанный хук — это технический долг.

## Настройка Husky для управления хуками

Husky — популярный инструмент для управления Git хуками. Использование Husky делает --no-verify менее привлекательной, так как хуки становятся предсказуемыми:

```bash
# Установить Husky
npm install husky --save-dev
npx husky install

# Добавить pre-commit хук
npx husky add .husky/pre-commit "npm run lint"

# Добавить commit-msg хук (проверка формата сообщения)
npx husky add .husky/commit-msg "npx commitlint --edit $1"

# Структура проекта
.husky/
  ├── pre-commit      # запускается перед коммитом
  ├── commit-msg      # проверяет сообщение коммита
  └── _/husky.sh      # вспомогательный скрипт

# При попытке git commit --no-verify эти проверки пропустятся
# Лучше настроить хуки правильно чем полагаться на --no-verify
```

## Настройка lint-staged для частичных проверок

`lint-staged` проверяет только изменённые файлы, что ускоряет процесс:

```bash
# Установить
npm install lint-staged --save-dev

# Конфигурация в package.json
{
  "lint-staged": {
    "*.js": ["eslint --fix", "git add"],
    "*.css": ["stylelint --fix", "git add"],
    "*.json": ["prettier --write", "git add"]
  }
}

# Добавить в pre-commit хук
npx husky add .husky/pre-commit "lint-staged"

# Теперь при commit будут проверяться только изменённые файлы
# Быстрее и не требует --no-verify
```

## Когда легальное использование --no-verify

Честные и оправданные причины:

```bash
# 1. Хук временно сломан
git commit --no-verify -m "WIP: save while fixing hook"
# Немедленно исправить хук!

# 2. Миграция из старой системы контроля версий
git commit --no-verify -m "Initial commit from SVN"

# 3. Автоматизированный процесс (CI)
# Сервер часто скипает хуки так как уже есть серверные проверки

# 4. Срочный hotfix в production
# Более поздний коммит может содержать правильный код
git commit --no-verify -m "HOTFIX: revert broken deployment"

# Но даже в этом случае — лучше исправить потом
git commit -m "fix: properly handle the issue"
```

## Собственный скрипт для выборочного пропуска

Вместо полного `--no-verify`, можно сделать хук умнее:

```bash
# .husky/pre-commit
#!/bin/bash

# Проверить переменную окружения
if [ "$SKIP_PRE_COMMIT" = "true" ]; then
    exit 0
fi

# Иначе запустить нормальные проверки
npm run lint

# Использование:
SKIP_PRE_COMMIT=true git commit -m "WIP: skip for now"

# Или создать алиас
git config --global alias.skip-hook '!SKIP_PRE_COMMIT=true git commit'
git skip-hook -m "WIP: temporary skip"
```

Это лучше чем `--no-verify` — видно что пропуск был намеренным.

## Policy для команды

Опытные команды устанавливают правила:

```
Правило 1: --no-verify можно использовать максимум 2 раза в месяц
Правило 2: Требует комментария в коммите почему пропущены хуки
Правило 3: Серверная проверка может заблокировать push

Формат сообщения с пропуском:
git commit --no-verify -m "NOQA: reason for skipping checks"

NOQA = No Questions Asked, но причину всё равно указать
```

Серверные хуки (`pre-receive`) гарантируют что даже с `--no-verify` некоторые проверки пройдут.

## Отладка когда хук блокирует работу

Если хук неправильно работает:

```bash
# 1. Включить verbose режим
npm run lint -- --debug
# или для конкретного хука

# 2. Запустить хук вручную
bash .husky/pre-commit
# Увидеть точную ошибку

# 3. Временно отключить и исправить
git commit --no-verify -m "fix: repair linter config"

# 4. Запушить исправление
git push origin main

# 5. Другим разработчикам:
git pull
# Хук снова начнёт работать правильно
```

## Часто задаваемые вопросы

**Можно ли обойти серверные хуки через --no-verify?** Нет. `--no-verify` отключает только клиентские хуки (`pre-commit`, `commit-msg`, `pre-push`). Серверные хуки (`pre-receive`, `update`, `post-receive`) выполняются на сервере и не зависят от клиентских флагов.

**Как настроить хуки так чтобы их нельзя было обойти?** Только через серверные хуки. Или через CI/CD — пайплайн проверяет код при каждом push независимо от хуков. Комбинация: клиентские хуки для быстрой обратной связи + серверные для гарантии.

**Остаётся ли след использования --no-verify?** В git log нет явного указания. Косвенно можно заметить по нарушениям в сообщениях коммитов или проблемам в коде. Серверный `pre-receive` хук может добавить проверку на стороне сервера. Лучше документировать почему пропуск был необходим.

**Что делать если хук сломался и блокирует работу?** Временно используйте `--no-verify`, но немедленно исправьте хук. Не игнорируйте — сломанный хук — это технический долг. Создайте task в issue tracker для исправления.

**Можно ли сделать разные правила для разных веток?** Да, через условную логику в хуке:
```bash
BRANCH=$(git symbolic-ref --short HEAD)
if [[ $BRANCH == "main" || $BRANCH == "develop" ]]; then
    npm run strict-lint
else
    npm run lint  # более мягкие проверки для feature веток
fi
```

**Нужны ли клиентские хуки если есть CI/CD?** Да! Клиентские хуки дают немедленную обратную связь. CI/CD может быть медленнее. Комбинируйте оба подхода.

## Заключение

`git commit --no-verify` пропускает `pre-commit` и `commit-msg` хуки. Оправданно использовать для срочных hotfix или при сломанном хуке, но только временно. Не стоит использовать регулярно — это разрушает систему автоматических проверок. Серверные хуки нельзя обойти клиентским `--no-verify`. Лучшая альтернатива — настроить хуки правильно через Husky и lint-staged. Для критических проверок используйте серверные хуки и CI/CD. Подробнее о создании хуков — [Git хуки]({{< relref "git-hooks" >}}).
