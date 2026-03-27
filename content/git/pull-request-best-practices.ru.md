---
title: "Pull Request: лучшие практики создания и проверки"
description: "Лучшие практики Pull Request: размер PR, описание, процесс ревью, стиль комментариев, автоматизация. Как делать эффективное code review."
date: 2026-02-28
lastmod: 2026-02-28
draft: false
slug: "pull-request-best-practices"
keywords: ["pull request best practices", "code review git", "как делать code review", "как создать pull request", "pull request описание", "правила pull request"]
tags: ["git", "workflow", "intermediate"]
categories: ["git"]
---

Pull Request (PR) — не просто кнопка слияния кода. Это инструмент коммуникации, документации и проверки качества. Хорошо составленный PR ускоряет review, а культура уважительного code review улучшает командную работу.

## Размер Pull Request

Маленькие PR рецензируются быстрее и лучше:

```
Оптимальный размер:
- < 200 строк изменений — идеально
- 200-400 строк — приемлемо
- > 400 строк — стоит разбить

Исследования показывают: ревьюеры эффективны
до 400 строк. Выше этого — качество падает,
проблемы пропускаются.

Исключения: автоматически сгенерированный код,
рефакторинг с переименованием, переводы.
```

**Как разбить большой PR:**

```bash
# Вместо одного большого PR:
# PR #1: Добавить API endpoint (backend)
# PR #2: Добавить UI форму (frontend)
# PR #3: Интегрировать API с UI

# Каждый PR независим и deployable
# (используйте feature flags для незавершённого)
```

## Описание Pull Request

Хорошее описание экономит время ревьюера:

```markdown
## Что сделано
Реализована двухфакторная аутентификация (2FA) через TOTP.

## Зачем
Пользователи запрашивали 2FA (#123). Повышает безопасность аккаунтов.

## Как тестировать
1. Войти в аккаунт
2. Перейти в Settings → Security
3. Нажать "Enable 2FA"
4. Отсканировать QR-код в Google Authenticator
5. Ввести 6-значный код
6. Убедиться что 2FA включена

## Скриншоты
[скриншот настройки 2FA]

## Технические детали
- Используется библиотека `speakeasy` для TOTP
- Секрет шифруется перед хранением (AES-256)
- Добавлены backup codes (10 одноразовых кодов)

## Что не входит в этот PR
- SMS 2FA (следующий PR)
- Hardware key (U2F) поддержка

Closes #123
```

## Создание PR через CLI

```bash
# GitHub CLI
gh pr create \
  --title "feat: add two-factor authentication" \
  --body "$(cat pr-template.md)" \
  --reviewer alice,bob \
  --label "security,feature"

# GitLab CLI
glab mr create \
  --title "feat: add 2FA" \
  --description "..." \
  --target-branch main \
  --assignee @alice

# Посмотреть список PR
gh pr list
gh pr status

# Проверить PR локально
gh pr checkout 123
```

## Процесс ревью

**Для ревьюера:**

```
1. Понять контекст (прочитать описание, связанные issues)
2. Посмотреть diff целиком
3. Запустить код локально если нужно
4. Оставить комментарии
5. Не затягивать ревью (< 24 часов)
```

**Приоритеты при ревью:**

```
🔴 Критично (блокирующее):
- Баги и логические ошибки
- Уязвимости безопасности
- Нарушение архитектурных принципов

🟡 Важно (обсудить):
- Нечитаемый код
- Производительность (с замерами)
- Дублирование

🟢 Нитпики (опционально):
- Стиль (если нет автоформатера)
- Имена переменных
- Мелкие улучшения
```

## Стиль комментариев

Конструктивный ревью строится на уважении:

```
Плохо:
"Это неправильно."
"Зачем ты так написал?"
"Перепиши."

Хорошо:
"Здесь возможна ошибка при пустом массиве —
 можно добавить проверку: if (arr.length === 0) return []"

"Рассматривал ли вариант с Map? Может быть эффективнее
 при большом количестве элементов. Но текущий вариант
 тоже работает — на твоё усмотрение."

"nit: можно заменить на деструктуризацию:
 const { id, name } = user"
```

**Prefix-маркировка комментариев:**

```
BLOCKER: — мешает merge, нужно исправить
QUESTION: — вопрос, нужен ответ
SUGGESTION: — предложение, необязательно
NIT: — мелочь, не блокирует
PRAISE: — отметить хорошее решение
```

## Шаблоны PR

Создайте шаблон для репозитория:

```markdown
<!-- .github/pull_request_template.md -->
## Тип изменений
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation

## Описание
<!-- Что сделано и зачем -->

## Тестирование
- [ ] Добавлены unit тесты
- [ ] Проверено вручную
- [ ] Существующие тесты проходят

## Checklist
- [ ] Код следует стандартам проекта
- [ ] Self-review выполнен
- [ ] Документация обновлена (если нужно)

Closes #
```

## Автоматизация review

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm test -- --coverage

  size-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

## CODEOWNERS

```
# .github/CODEOWNERS
# Автоматически назначает ревьюеров по папкам

# Вся документация → docs team
docs/          @docs-team

# Security-критичные файлы
src/auth/      @security-team @lead-dev
.env.example   @security-team

# Frontend
src/components/  @frontend-team

# Backend API
src/api/         @backend-team

# Любые изменения → lead developer
*                @lead-dev
```

## Часто задаваемые вопросы

**Можно ли merge PR без ревью?** Технически да, но не рекомендуется. Настройте branch protection: Settings → Branches → Require pull request reviews. Для срочных hotfix — можно упростить, но ревью после merge.

**Сколько ревьюеров нужно?** Зависит от команды. Обычно 1-2 ревьюера. Больше — не всегда лучше (diffusion of responsibility). Для критичных изменений (безопасность, архитектура) — senior разработчик обязателен.

**Как реагировать на критику в ревью?** Критика кода — не критика человека. Если не согласны — объясните своё решение. Если ревьюер прав — поблагодарите и исправьте. Если не уверены — обсудите в чате.

**Нужны ли PR для маленьких изменений?** Зависит от соглашения команды. Многие используют draft PR даже для мелких изменений — для трассируемости. Для исправления опечатки в документации — иногда прямой commit в main.

## Заключение

Хороший PR: маленький (< 400 строк), с понятным описанием, с тестами и самопроверкой. Хорошее ревью: конструктивное, своевременное (< 24ч), с приоритизацией комментариев. Культура уважительного code review — основа продуктивной команды. Автоматизируйте проверки через CI/CD, используйте CODEOWNERS и PR шаблоны.

## По теме

- [GitHub Flow]({{< relref "github-flow" >}})
- [Gitflow]({{< relref "gitflow" >}})
- [Форк репозитория]({{< relref "fork-repozitoriya" >}})
- [Лучшие практики коммитов]({{< relref "luchshie-praktiki-kommitov" >}})
- [Conventional Commits]({{< relref "conventional-commits" >}})
- [Feature Branch Workflow]({{< relref "feature-branch-workflow" >}})
