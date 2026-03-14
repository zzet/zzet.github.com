---
title: "Git Workflow: как выбрать модель ветвления для команды"
description: "Обзор популярных Git workflow. Сравнение Gitflow, GitHub Flow, Trunk-Based Development. Как выбрать подходящую модель для вашей команды."
date: 2026-02-01
lastmod: 2026-02-01
draft: false
slug: "git-workflow"
keywords: ["git workflow", "git модель ветвления", "какой workflow выбрать", "git branching strategy", "какую модель ветвления выбрать", "git flow vs trunk based"]
tags: ["git", "workflow", "intermediate"]
categories: ["git"]
---

Git workflow — это соглашение команды о том, как использовать ветки, когда коммитить и как деплоить. Правильный workflow упрощает совместную разработку и снижает количество конфликтов. Неправильный — создаёт накладные расходы без пользы.

## Что такое Git Workflow

Workflow определяет:

```
- Сколько долгоживущих веток существует
- Как создаются и называются ветки задач
- Когда и как происходит merge
- Кто проверяет код (code review)
- Как связан git с деплоем
```

Единого правильного ответа нет — выбор зависит от размера команды, частоты релизов, зрелости CI/CD.

## Основные модели

### Centralized Workflow (для перехода с SVN)

```bash
# Один trunk, все коммитят напрямую
# Для маленьких команд или при переходе с SVN

git switch main
git pull --rebase origin main
git commit -am "Add feature"
git push origin main
```

Когда подходит: команда из 2-3 разработчиков, простой проект.

### Feature Branch Workflow

```bash
# Основа большинства workflow
# Каждая задача — своя ветка

git switch -c feature/user-profile
# ... разработка ...
git push origin feature/user-profile
# → Pull Request → Review → Merge
```

Когда подходит: большинство команд, любой размер.

### GitHub Flow

```bash
# main всегда deployable
# Feature ветки от main, PR для review

git switch -c feature/dark-mode
git push origin feature/dark-mode
gh pr create
# CI/CD деплоит после merge в main
```

Когда подходит: SaaS, web-приложения, непрерывный деплой.

### Gitflow

```bash
# Структурированные релизы
# main + develop + feature/release/hotfix

git flow feature start user-auth
# ... разработка ...
git flow feature finish user-auth
git flow release start 1.2.0
git flow release finish 1.2.0
```

Когда подходит: приложения с плановыми релизами, мобильные приложения.

### Trunk-Based Development

```bash
# Коммиты в main несколько раз в день
# Feature flags для незавершённого кода

git switch main
git commit -am "feat: add search (behind flag)"
git push origin main
# CI/CD сразу тестирует и деплоит
```

Когда подходит: опытные команды, высокая частота деплоев.

## Сравнение workflow

```
Параметр          Centralized  Feature   GitHub    Gitflow   TBD
                              Branch    Flow
───────────────────────────────────────────────────────────────────
Сложность         Минимальная  Базовая   Базовая   Высокая   Средняя
Ветки             1            2+        2+        5+        1-2
Деплой            Ручной       По ветке  Auto/main По тегу   Auto/main
Version mgmt      Нет          Нет       Тегами    Полное    Тегами
Code review       Нет          PR        PR        PR        PR/Direct
Подходит для      Мигр. SVN    Большинство  CD     Релизы    Опытные
```

## Ключевые компоненты любого workflow

**Защита main ветки:**

```
GitHub: Settings → Branches → Add rule
Требования:
✓ Pull request перед merge
✓ Required reviewers (1-2 человека)
✓ Status checks (CI должен пройти)
✓ Branch up to date (актуальная ветка)
```

**Соглашение об именовании:**

```
feature/short-description
fix/bug-description
hotfix/critical-issue
release/1.2.0
chore/update-dependencies
```

**CI/CD интеграция:**

```yaml
# .github/workflows/ci.yml (GitHub Actions)
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm test
      - run: npm run lint
```

## Как выбрать workflow

**Задайте команде вопросы:**

```
1. Как часто нужно деплоить?
   - Несколько раз в день → TBD или GitHub Flow
   - Раз в неделю/месяц → Gitflow или GitHub Flow

2. Сколько версий в production?
   - Одна версия → GitHub Flow или TBD
   - Несколько версий → Gitflow

3. Какой размер команды?
   - 1-5 разработчиков → Feature Branch или GitHub Flow
   - 5-20 → GitHub Flow или Gitflow
   - 20+ → TBD или масштабированный Gitflow

4. Насколько зрелый CI/CD?
   - Хороший CI/CD → GitHub Flow или TBD
   - Базовый или нет → Gitflow (больше ручного контроля)
```

## Автоматизация workflow

```bash
# GitHub Actions: автодеплой при merge в main
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: ./deploy.sh production

# Автоматическое удаление веток при merge
# GitHub Settings → Delete branch on merge

# Commitlint для стандарта коммитов
npm install -D @commitlint/cli @commitlint/config-conventional
echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js
```

## Часто задаваемые вопросы

**Обязательно ли следовать одному workflow?** Нет. Можно адаптировать. Главное — договорённость команды и последовательное применение.

**Может ли workflow меняться со временем?** Да. Начните с простого (Feature Branch), переходите к сложному по мере роста.

**Что делать если workflow не работает?** Ретроспектива: что именно не работает, почему. Часто проблема не в workflow, а в CI/CD, тестах или коммуникации.

## Детальное объяснение Feature Branch Workflow

Feature Branch Workflow — наиболее универсальный подход:

```bash
# 1. Создать feature ветку от main
git switch main
git pull --rebase origin main
git switch -c feature/user-authentication

# 2. Разрабатывать на своей ветке
git add src/auth.js
git commit -m "feat: implement login form"
git add tests/auth.test.js
git commit -m "test: add authentication tests"

# 3. Отправить на GitHub
git push origin feature/user-authentication

# 4. Создать Pull Request на GitHub
# GitHub UI: Compare & pull request

# 5. Code Review — товарищи смотрят код
# - Обсуждение в комментариях PR
# - Запросы на изменения (Request Changes)
# - Утверждение (Approve)

# 6. После утверждения — merge в main
# GitHub UI: Merge pull request

# 7. После merge удалить feature ветку
git switch main
git pull origin main
git branch -d feature/user-authentication
git push origin --delete feature/user-authentication

# 8. Все остальные обновляют main
git switch main
git pull origin main
```

Этот процесс обеспечивает quality gates и知识 sharing в команде.

## Gitflow: детальная инструкция для релизов

Gitflow используется когда релизы планируются заранее:

```bash
# Инициализировать gitflow
git flow init
# Вопросы про имена веток — можно оставить по умолчанию

# Начать feature
git flow feature start user-profile
# Создаст ветку feature/user-profile от develop

# Разработка
git add src/profile.js
git commit -m "feat: user profile page"
git push origin feature/user-profile

# Закончить feature
git flow feature finish user-profile
# Merge в develop и удаление feature ветки

# Подготовка к релизу (когда features готовы)
git flow release start 1.2.0
# Создаст ветку release/1.2.0 от develop

# Финальные правки в release ветке
echo "1.2.0" > version.txt
git commit -am "bump: version 1.2.0"

# Закончить release
git flow release finish 1.2.0
# Merge в main + develop, создаст тег 1.2.0

# Критический баг в production — hotfix
git flow hotfix start 1.2.1
# Создаст ветку hotfix/1.2.1 от main

# Исправить баг
git add src/bug.js
git commit -m "fix: critical security issue"

# Закончить hotfix
git flow hotfix finish 1.2.1
# Merge в main и develop
```

## Trunk-Based Development для опытных команд

TBD требует высокой дисциплины и хорошего CI/CD:

```bash
# Вместо long-lived feature веток — краткосрочные ветки
# Коммиты в main несколько раз в день

# Для незавершённого функционала — feature flags
// src/features.js
const FEATURES = {
  newDashboard: process.env.FEATURE_NEW_DASHBOARD === 'true',
  betaSearch: process.env.FEATURE_BETA_SEARCH === 'true',
}

// src/dashboard.js
import { FEATURES } from './features'

export function Dashboard() {
  if (FEATURES.newDashboard) {
    return <NewDashboard />
  }
  return <OldDashboard />
}

# Сам код в main, но можно отключить

# Рабочий процесс:
git switch -c feature/new-dashboard  # краткоживущая ветка
# ... разработка ...
git commit -m "feat: new dashboard behind flag"
git push origin feature/new-dashboard
# PR и merge в main

# После merge и deploy флаг включается постепенно:
FEATURE_NEW_DASHBOARD=true git push  # плавное включение для % пользователей
```

TBD требует: хороший CI/CD, feature flags, хорошие тесты, частый деплой.

## Выбор workflow по размеру команды

```
Размер: 1-3 человека
├─ Вариант 1: Centralized (коммиты прямо в main)
└─ Вариант 2: Feature Branch (если планируют долго работать)

Размер: 3-10 человек
├─ Feature Branch Workflow (стандарт)
├─ + обязательный code review
└─ + protected main branch

Размер: 10-30 человек
├─ GitHub Flow (если частый деплой)
├─ Gitflow (если плановые релизы)
└─ Много PR in-flight одновременно

Размер: 30+ человек
├─ TBD с feature flags
├─ или масштабированный Gitflow (по модулям)
└─ Strong CI/CD обязателен
```

## Workflow для open source (fork + PR)

Для внешних контрибьютеров:

```bash
# На GitHub: нажать Fork (создаёт копию под вашим аккаунтом)

# Клонировать свой форк
git clone https://github.com/YOUR_USERNAME/project.git
cd project

# Добавить upstream (исходный репозиторий)
git remote add upstream https://github.com/ORIGINAL_OWNER/project.git

# Создать feature ветку
git switch -c feature/fix-typo

# Разработка и коммиты
git commit -am "fix: typo in README"

# Отправить в ваш форк
git push origin feature/fix-typo

# На GitHub: нажать "Create Pull Request"
# PR будет отправлен в исходный репозиторий

# Синхронизация с upstream если долго разрабатываете
git fetch upstream
git rebase upstream/main

git push -f origin feature/fix-typo
```

## Workflow в monorepo

Когда несколько приложений в одном репо:

```bash
# Структура
project/
  ├─ apps/
  │  ├─ web/
  │  ├─ mobile/
  │  └─ api/
  └─ packages/
     └─ shared-utils/

# Feature ветки могут затрагивать несколько приложений
git switch -c feature/api-v2-and-web-update

# Коммиты касаются разных частей
git add apps/api/v2/handler.js
git commit -m "feat(api): implement v2 endpoint"

git add apps/web/src/fetch.js
git commit -m "feat(web): consume v2 API"

git add packages/shared-utils/request.js
git commit -m "feat(shared): add v2 request helper"

# Один PR но несколько изменений
# CI/CD обычно тестирует все затронутые приложения
```

## Часто задаваемые вопросы

**Обязательно ли следовать одному workflow?** Нет. Можно адаптировать. Главное — договорённость команды и последовательное применение.

**Может ли workflow меняться со временем?** Да. Начните с простого (Feature Branch), переходите к сложному по мере роста.

**Что делать если workflow не работает?** Ретроспектива: что именно не работает, почему. Часто проблема не в workflow, а в CI/CD, тестах или коммуникации.

**Как выбрать между GitHub Flow и Gitflow?** GitHub Flow если деплоим в main часто (каждый день). Gitflow если планируем релизы заранее (раз в месяц/квартал).

**Нужен ли code review в маленькой команде?** Полезен. Даже в двухчеловеческой команде code review ловит ошибки и распространяет знание.

**Может ли быть несколько main веток?** В Gitflow есть main (продакшен) и develop (разработка). В других workflow — обычно одна main. Несколько main веток создают путаницу.

**Как долго жить feature ветке?** Идеально: несколько дней. Если дольше 1-2 недель — может быть конфликт при merge.

## Заключение

Не существует универсального «лучшего» workflow. Для большинства команд Feature Branch Workflow с Pull Requests — хорошая отправная точка. Для непрерывного деплоя — [GitHub Flow]({{< relref "github-flow" >}}). Для плановых релизов — [Gitflow]({{< relref "gitflow" >}}). Для высокоскоростных команд — [Trunk-Based Development]({{< relref "trunk-based-development" >}}). Главное — договорённость команды и автоматизация. Выбирайте workflow исходя из размера команды, частоты релизов и зрелости CI/CD.
