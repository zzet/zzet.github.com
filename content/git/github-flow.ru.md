---
title: "GitHub Flow: простой workflow для непрерывного деплоя"
description: "Руководство по GitHub Flow. Одна ветка main, feature ветки и Pull Requests. Когда выбрать GitHub Flow вместо Gitflow. Автоматизация через GitHub Actions."
date: 2026-02-03
lastmod: 2026-02-03
draft: false
slug: "github-flow"
keywords: ["github flow", "github flow workflow", "github flow vs gitflow", "github flow ветки", "github flow pull request", "github flow vs git flow"]
tags: ["git", "workflow", "beginner"]
categories: ["git"]
---

GitHub Flow — упрощённая модель ветвления, предложенная GitHub в 2011 году. В отличие от Gitflow, здесь только одна долгоживущая ветка (`main`) и короткоживущие feature ветки. Идеально для команд с непрерывным деплоем.

## Принципы GitHub Flow

```
1. main всегда deployable
   — всё в main прошло review и готово к деплою

2. Новая работа — новая ветка от main
   — feature, fix, experiment — всё в отдельной ветке

3. Коммиты в ветку регулярно
   — backup, обсуждение, CI feedback

4. Pull Request → обсуждение и review
   — код не сливается без review

5. После approve — деплой из ветки
   — можно проверить в staging перед merge

6. Merge в main
   — только после успешного деплоя/тестирования
```

## Процесс разработки

**Шаг 1: Создать ветку от main**

```bash
# Обновить main
git switch main
git pull origin main

# Создать feature ветку
git switch -c feature/user-profile
# или bugfix
git switch -c fix/login-error
```

**Шаг 2: Разработка с регулярными коммитами**

```bash
git commit -am "feat: add profile page layout"
git commit -am "feat: implement avatar upload"
git commit -am "test: add profile page tests"

# Регулярно пушить для backup и CI
git push origin feature/user-profile
```

**Шаг 3: Открыть Pull Request**

```bash
# Через GitHub CLI
gh pr create --title "Add user profile page" \
  --body "Implements profile editing and avatar upload. Closes #45"

# Или через GitHub UI:
# Compare & pull request → Fill title & description → Create PR
```

**Шаг 4: Обсуждение и review**

```bash
# Внести изменения по комментариям ревьюеров
git commit -am "fix: address review comments"
git push origin feature/user-profile

# Ответить на комментарии в GitHub UI
```

**Шаг 5: Деплой и проверка**

```bash
# GitHub Actions автоматически деплоит ветку в staging
# или вручную:
# Merge в staging ветку → проверить → merge в main
```

**Шаг 6: Merge в main**

```bash
# После approve и прохождения CI:
gh pr merge --squash  # через GitHub CLI
# или через GitHub UI: Merge pull request
```

## Автоматизация через GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
      - run: npm ci
      - run: npm test
      - run: npm run lint

  deploy-preview:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy preview
        run: ./deploy-preview.sh ${{ github.head_ref }}

  deploy-production:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to production
        run: ./deploy.sh production
```

## Именование веток в GitHub Flow

```bash
# Рекомендуемые паттерны:
feature/короткое-описание      # feature/add-search
fix/что-исправляем             # fix/login-redirect
docs/что-документируем         # docs/api-authentication
chore/что-делаем               # chore/update-dependencies
experiment/идея                # experiment/new-ui-concept

# Реальные примеры:
git switch -c feature/dark-mode
git switch -c fix/null-pointer-homepage
git switch -c chore/upgrade-react-18
git switch -c docs/deployment-guide
```

## GitHub Flow vs Gitflow

```
GitHub Flow:
+ Простой (только main + feature ветки)
+ Непрерывный деплой из main
+ Подходит для SaaS и web-сервисов
+ Меньше процесса — больше скорости
- Нет механизма для hotfix отдельно от разработки
- Нет версионных релизов (все в main)

Gitflow:
+ Чёткие релизные версии
+ Поддержка нескольких версий
+ Hotfix не смешивается с разработкой
- Сложнее (5 типов веток)
- Медленнее (больше шагов)
- Избыточен для continuous deployment
```

## Protected branches

```
Рекомендуемые правила защиты main в GitHub:
- Require pull request before merging (обязателен PR)
- Require status checks (CI должен пройти)
- Require branches to be up to date (ветка актуальна)
- Restrict who can push to main (только через PR)
- Require review from Code Owners (для критических файлов)
```

Настройка: Settings → Branches → Add rule.

## Развёртывание из feature ветки перед merge

Один из ключевых преимуществ GitHub Flow — возможность тестировать feature ветку в production-подобной среде (staging) перед мержем в main:

```bash
# После создания PR, можно развернуть feature ветку на staging
git push origin feature/user-profile

# GitHub Actions может автоматически создать deployment preview
# или вручную через GitHub UI: Environments → Create deployment
```

Пример workflow для deployment preview:

```yaml
# .github/workflows/deploy-preview.yml
name: Deploy Preview

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy to preview environment
        run: |
          PREVIEW_URL="https://preview-${{ github.head_ref }}.app.example.com"
          ./scripts/deploy-preview.sh "$PREVIEW_URL"

      - name: Comment PR with preview link
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Preview deployed: https://preview-${{ github.head_ref }}.app.example.com'
            })
```

## GitHub Flow в одиночной разработке

GitHub Flow работает хорошо даже для одного разработчика:

```bash
# Разработчик работает в feature ветке (даже если один)
git switch -c feature/dark-mode
git commit -am "feat: add dark mode toggle"
git commit -am "feat: persist dark mode preference"
git push origin feature/dark-mode

# Сам себе делает PR для self-review
gh pr create --title "Add dark mode support"

# Проверяет свой собственный diff через PR UI
# (нередко при просмотре собственного PR замечаются ошибки)

# После self-review — merge
gh pr merge --squash
git switch main
git pull origin main
```

Даже solo разработчик получает преимущества:
- История веток сохраняется
- Можно просмотреть diff перед merge
- CI запускается автоматически
- Легко откатиться если нужно

## GitHub Flow в команде: процесс code review

В команде GitHub Flow требует структурированного code review:

```bash
# Разработчик А создаёт PR
git switch -c feature/payment-integration
# ... работа ...
git push origin feature/payment-integration
gh pr create --title "Integrate Stripe payments" \
  --body "Adds payment processing via Stripe.

  Related: #145
  Tests added: tests/payment.test.js
  Staging link: https://staging-feat.app.com"
```

Рекомендуемый чеклист для code review:

```markdown
# Code Review Checklist

## Функциональность
- [ ] Код делает то, что задумано
- [ ] Обработаны edge cases
- [ ] Нет очевидных ошибок

## Качество кода
- [ ] Следует style guide
- [ ] Нет dead code
- [ ] Переменные названы понятно

## Тесты
- [ ] Добавлены/обновлены тесты
- [ ] Тесты проходят локально
- [ ] Покрытие достаточно

## Безопасность
- [ ] Нет hardcoded секретов
- [ ] Валидация input
- [ ] Аутентификация/авторизация

## Performance
- [ ] Нет N+1 queries
- [ ] Оптимизированы медленные операции
```

## Таблица сравнения: GitHub Flow vs Gitflow vs Trunk-Based Development

```
Характеристика          GitHub Flow         Gitflow           Trunk-Based Dev
─────────────────────────────────────────────────────────────────────────
Долгожив. ветки         1 (main)           2 (main, dev)      1 (main)
Feature ветки           1-2 недели         2-4 недели         < 1-2 дня
Версионирование         Через теги         Через release      Через теги
Hotfix процесс          Как обычная PR      Отдельная ветка    Обычный PR
Требует feature flags   Опционально        Нет                Обязательно
Частота деплоя          Несколько раз/день Раз в неделю/месяц  Несколько раз/день
CI/CD сложность         Средняя            Низкая             Высокая
Подходит для            SaaS, web          Desktop, mobile    Опытные команды
Merge конфликты         Редко              Часто (dev ветка)  Очень редко
─────────────────────────────────────────────────────────────────────────
```

## Ветки для разных типов работ

GitHub Flow рекомендует стандартные префиксы:

```bash
# Feature: новая функциональность
git switch -c feature/oauth-github

# Bugfix: исправление ошибок
git switch -c bugfix/null-pointer-dashboard

# Improvement: улучшение существующего
git switch -c improvement/pagination-performance

# Refactor: рефакторинг без изменения функциональности
git switch -c refactor/auth-module

# Documentation: обновление docs
git switch -c docs/api-authentication

# Chore: dependency update, build config
git switch -c chore/upgrade-webpack

# Experiment: экспериментальное
git switch -c experiment/new-ui-framework
```

## Когда GitHub Flow НЕ подходит

GitHub Flow хороший выбор для большинства, но есть случаи когда лучше Gitflow:

```
Используйте GitHub Flow:
✓ SaaS приложения (непрерывный деплой)
✓ Web сервисы (многоразовый деплой)
✓ Опытные разработчики
✓ Быстрый цикл разработки

НЕ используйте GitHub Flow:
✗ Мобильные приложения (плотный review process Apple/Google)
✗ Библиотеки с версионированием
✗ Приложения с плановыми релизами
✗ Нужны разные версии в support одновременно
✗ Очень большие команды (100+ разработчиков)

В этих случаях рассмотрите [Gitflow]({{< relref "gitflow" >}}) или
[Trunk-Based Development]({{< relref "trunk-based-development" >}})
```

## Практические советы по GitHub Flow

```bash
# 1. Обновлять feature ветку перед push (избежать конфликтов)
git switch main
git pull origin main
git switch feature/new-api
git rebase main
git push origin feature/new-api --force-with-lease

# 2. Разбивать большие PR на несколько меньших
# Лучше 3 PR по 100 строк, чем 1 PR по 300 строк

# 3. Использовать Conventional Commits для автоматического changelog
git commit -m "feat: add user profile endpoint"
git commit -m "fix: handle null avatar URL"
git commit -m "docs: update API docs for profiles"
# Потом: npm run release (автоматически генерирует CHANGELOG)

# 4. Если PR слишком долгий, разделить на части
# Не ждать полного завершения, мержить готовые части

# 5. Защищать main от случайных push
# Settings → Branches → Add rule for 'main'
```

## Deploy Previews и Feature Flags с GitHub Flow

Современные платформы поддерживают автоматические deploy previews:

```yaml
# Vercel автоматически создаёт preview для каждого PR
# Просто push в GitHub — готова staging среда
# URL: https://project-name-git-feature-branch-name.vercel.app

# Для самоходного хоста можно использовать Netlify, Render, Railway
# или собственное решение через GitHub Actions
```

Feature flags для gradual rollout:

```javascript
// Деплоим feature в production но скрытую от пользователей
const featureFlags = {
  newCheckout: localStorage.getItem('ff_new_checkout') === 'true' ||
               userId in [123, 456, 789], // или по процентам
};

if (featureFlags.newCheckout) {
  return <NewCheckoutFlow />;
}
return <LegacyCheckout />;
```

## Часто задаваемые вопросы

**Что делать с hotfix в GitHub Flow?** GitHub Flow не выделяет hotfix как особый тип. Просто создать ветку от main: `git switch -c fix/critical-security-patch`. Если нужно срочно — PR с пометкой "urgent" и упрощённый review.

**Как управлять версиями без release веток?** Через теги: `git tag -a v1.2.0 -m "Release 1.2.0" && git push --tags`. Для генерации changelog — Semantic Release или Conventional Commits.

**Можно ли deploy из feature ветки?** Да, это часть GitHub Flow — деплоить feature ветку в staging для проверки перед merge в main. GitHub Actions поддерживает deployment environments для этого.

**Как часто должен происходить merge в main?** Как можно чаще. Feature ветки должны быть короткоживущими (1-3 дня). Длинные ветки приводят к сложным конфликтам.

## Заключение

GitHub Flow: `main` всегда deployable, новая работа — в feature ветке от main, PR для review, merge после проверки. Идеально для web-сервисов с непрерывным деплоем. Проще Gitflow, но без version management. Для релизов с версиями — рассмотрите [Gitflow]({{< relref "gitflow" >}}). Для максимальной скорости с опытной командой — [Trunk-Based Development]({{< relref "trunk-based-development" >}}).
