---
title: "Trunk-Based Development: разработка прямо в main"
description: "Что такое Trunk-Based Development (TBD). Принципы, feature flags, сравнение с Gitflow. Как перейти на TBD и что требуется от команды."
date: 2026-03-07
lastmod: 2026-03-07
draft: false
slug: "trunk-based-development"
keywords: ["trunk-based development", "TBD git workflow", "trunk based development vs gitflow", "trunk based development что это", "разработка в одной ветке git", "main branch only development"]
tags: ["git", "workflow", "advanced"]
categories: ["git"]
---

Trunk-Based Development (TBD) — модель, при которой все разработчики регулярно (несколько раз в день) коммитят напрямую в основную ветку (trunk/main). Feature flags скрывают незавершённые функции от пользователей. Используется Google, Facebook, Netflix.

## Принципы TBD

```
1. Один trunk (main) — единственная долгоживущая ветка
2. Частые коммиты: не реже 1-2 раз в день в main
3. Короткие feature ветки (< 1-2 дней)
4. Feature flags для незавершённых функций
5. Автоматические тесты — обязательно
6. CI/CD — непрерывная интеграция и деплой
```

## TBD vs Gitflow vs GitHub Flow

```
                TBD          GitHub Flow    Gitflow
──────────────────────────────────────────────────────
Долгожив.
ветки           1 (main)     1 (main)       2 (main+develop)
Feature
ветки           < 2 дней     дни-недели     недели
Частота
коммитов        несколько/день  по задаче  по задаче
Feature
flags           обязательно  опционально    не нужны
CI/CD           обязательно  важно         опционально
Сложность       низкая       низкая        высокая
Подходит для    опытные      большинство   плановые релизы
                команды      команд
```

## Feature flags: основа TBD

Feature flag позволяет деплоить незавершённую функцию скрытой от пользователей:

```javascript
// Простая реализация feature flag
const featureFlags = {
  newDashboard: process.env.FEATURE_NEW_DASHBOARD === 'true',
  betaCheckout: process.env.FEATURE_BETA_CHECKOUT === 'true',
};

// Использование
function renderDashboard() {
  if (featureFlags.newDashboard) {
    return <NewDashboard />;
  }
  return <OldDashboard />;
}
```

```python
# Python пример
FEATURE_FLAGS = {
    'new_payment_flow': os.getenv('FF_NEW_PAYMENT', 'false') == 'true',
}

def process_payment(amount):
    if FEATURE_FLAGS['new_payment_flow']:
        return new_payment_processor(amount)
    return legacy_payment_processor(amount)
```

Флаг выключен в production → можно деплоить незавершённый код → включить когда готово.

## Рабочий процесс при TBD

**Маленькие задачи (< 1 дня):**

```bash
# Коммитим прямо в main
git switch main
git pull origin main

# Небольшое изменение
git commit -am "feat: add email validation to signup form"
git push origin main

# CI/CD автоматически тестирует и деплоит
```

**Большие задачи (> 1 дня):**

```bash
# Добавить feature flag
# Создать краткосрочную ветку
git switch -c feature/new-checkout
# Разработка максимум 1-2 дня
git commit -am "feat: implement new checkout flow (behind flag)"
git push origin feature/new-checkout

# Merge через PR как можно скорее
gh pr create --title "New checkout (behind feature flag)"
# После review (не более дня) — merge
```

## Требования для TBD

```
Технические требования:
✓ Развитая CI/CD система (тесты за < 10 минут)
✓ Feature flags система
✓ Высокое тестовое покрытие (>70-80%)
✓ Мониторинг и быстрый откат (canary deploys)
✓ Trunk всегда в deployable состоянии

Командные требования:
✓ Опытная команда (понимание рисков)
✓ Code review культура
✓ Доверие между разработчиками
✓ Договорённость о маленьких задачах
```

## Масштабированный TBD

Для больших команд:

```bash
# Краткосрочные feature ветки (1-2 дня максимум)
git switch -c feature/add-search-v2
# ... разработка в течение дня ...
git push origin feature/add-search-v2
gh pr create
# Immediate review → merge → delete branch

# Ветки релизов (только для чтения, нет разработки)
# Создаётся от конкретного коммита main для стабилизации
git switch -c release/1.5 main
# Только cherry-pick критических фиксов
git cherry-pick <hotfix-commit>
```

## Преимущества TBD

```
+ Быстрое обнаружение конфликтов интеграции
  (не накапливаются недели в длинных ветках)

+ Непрерывная интеграция работает по-настоящему
  (не "мы интегрируем раз в неделю")

+ Меньше merge конфликтов
  (код интегрируется часто, не копится)

+ Прозрачность прогресса
  (все видят реальное состояние в main)

+ Подходит для Continuous Deployment
```

## Недостатки TBD

```
- Требует дисциплины (неопытные разработчики ломают main)
- Feature flags усложняют код
  (нужно убирать флаги после выпуска функции)
- Сложнее управлять несколькими версиями
- Высокие требования к CI/CD
- Не подходит для open source (много контрибьюторов разного уровня)
```

## Переход на TBD

```bash
# Шаг 1: Настроить CI с автоматическими тестами
# Шаг 2: Внедрить feature flags систему
# Шаг 3: Начать с коротких feature веток (< 2 дней)
# Шаг 4: Постепенно сокращать время жизни веток
# Шаг 5: Лучшие разработчики начинают коммитить напрямую
# Шаг 6: Вся команда переходит на TBD

# Метрика успеха:
# - Среднее время жизни ветки < 1 дня
# - Коммиты в main: 5-10+ в день на разработчика
# - Время CI: < 10 минут
```

## TBD в крупных компаниях

Как используют TBD в Google, Facebook, Netflix:

**Google:**
```
- Все разработчики коммитят в единый репозиторий (monorepo)
- Тысячи разработчиков, десятки тысяч коммитов в день
- Обязательно: сильная CI (тесты за минуты)
- Feature flags широко используются для управления функциями
- Canary deploys (1% → 10% → 100% постепенно)
```

**Facebook:**
```
- Trunk-based development в core продуктов
- Deploy каждые 15 минут в production
- Strict code review перед merge
- Обязательное тестовое покрытие >70%
- Быстрое обнаружение и откат ошибок
```

**Netflix:**
```
- Микросервисная архитектура + TBD
- Chaos engineering для обнаружения проблем
- Очень быстрые тесты (< 5 минут)
- Independent deployments per service
- Feature flags для A/B тестирования
```

## TBD в CI/CD пайплайне

Пример полного пайплайна для TBD:

```yaml
# .github/workflows/tbd-pipeline.yml
name: TBD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v3

      # Быстрые юнит тесты
      - name: Run unit tests
        run: npm test -- --coverage

      # Lint и code quality
      - name: Lint
        run: npm run lint

      # Static analysis
      - name: SonarQube scan
        run: ./gradlew sonarqube

  integration-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Integration tests
        run: npm run test:integration

  deploy-staging:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: [test, integration-test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to staging
        run: ./scripts/deploy-staging.sh
      - name: Smoke tests on staging
        run: npm run test:smoke

  canary-deploy:
    if: github.event_name == 'push'
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to 1% of production
        run: ./scripts/canary-deploy.sh --percentage 1

      - name: Monitor metrics
        run: ./scripts/monitor.sh

      - name: If stable, promote to 100%
        run: ./scripts/full-deploy.sh
```

## Требования и предпосылки для TBD

Чтобы TBD работал, нужны:

**Технические требования:**

```bash
# 1. Быстрые тесты (< 10 минут для всей suite)
npm test                    # быстро!

# 2. Автоматическое тестовое покрытие
npm test -- --coverage
# Минимум 70-80% покрытия

# 3. Feature flags система
npm install @unleash/proxy-client  # или Launchdarkly
# const flags = unleash.isEnabled('new-dashboard')

# 4. Мониторинг и откат
# Alerting на метриках
# Автоматический откат при spike errors

# 5. Trunk всегда deployable
git log main --oneline -5
# Все 5 последних коммитов должны пройти CI
```

**Командные требования:**

```
- Опытные разработчики (понимают рисками)
- Культура code review (не skip review)
- Доверие в команде
- Договорённость: маленькие задачи (< 1 дня)
- Дежурство (кто-то мониторит deploy)
```

## Переход с Gitflow на TBD: пошаговый план

Миграция требует подготовки:

```bash
# Этап 1: Подготовить инфраструктуру (2-4 недели)
# - Настроить CI (GitHub Actions, GitLab CI, Jenkins)
# - Implement feature flags
# - Настроить мониторинг и alerting

# Этап 2: Внедрить shorter-lived branches (1-2 недели)
# - Обучить команду: ветки максимум 3 дня
# - Установить branch protection правила

# Этап 3: Пилотная группа (1-2 недели)
# - 3-4 разработчика начинают коммитить в main
git commit -am "feat: new API endpoint (behind flag)"
git push origin main
# Остаток команды → PR в feature ветке

# Этап 4: Расширение (2-3 недели)
# - Больше разработчиков в TBD
# - Постоянно собирается обратная связь

# Этап 5: Полный переход (1-2 недели)
# - Вся команда в TBD
# - Удалить develop ветку
git push origin --delete develop

# Этап 6: Оптимизация (ongoing)
# - Ускорить тесты ещё больше
# - Автоматизировать больше
# - Обучение новых членов команды
```

## Таблица: TBD vs GitHub Flow vs Gitflow

```
Аспект                  TBD                GitHub Flow        Gitflow
────────────────────────────────────────────────────────────────────────
Долгоживущие ветки      main               main               main, develop
Кол-во разработчиков    100+               5-50               10-100
Ветки интеграции        нет                нет                develop
Release ветки           по необходимости   нет                release/x.x
Hotfix ветки            нет                нет                hotfix/x.x
Частота коммитов        несколько/день     1-2/день           1/неделю
Feature флаги           ОБЯЗАТЕЛЬНО        опционально        редко
Merge конфликты         редко              редко              частые
Время ветки             < 1-2 дня          дни-недели         недели-месяцы
CI/CD требование        ОБЯЗАТЕЛЕН         высокий            средний
Тестовое покрытие       >70-80%           >50%               >40%
Rollback стратегия      feature flags      revert commit      hotfix branch
────────────────────────────────────────────────────────────────────────
```

## Objections и ответы на них

**Возражение: "Разработчики сломают main"**

Ответ:
```
- Вот почему нужны автоматические тесты (CI должен ловить ошибки)
- Code review перед merge (не пропускаем)
- Feature flags (сломанный код скрыт)
- Опытная команда (знает что делает)
- Обучение новичков (наставничество)
```

**Возражение: "Feature флаги усложняют код"**

Ответ:
```
- Временное усложнение (флаг удаляется после выпуска)
- Стоит того для:
  - Упрощения deployment
  - A/B тестирования
  - Постепенного rollout (canary deploy)
- Используйте управление флагами (Unleash, LaunchDarkly)
```

**Возражение: "Нельзя поддерживать несколько версий"**

Ответ:
```
- TBD не для библиотек со строгим версионированием
- Если нужна поддержка 2 versions одновременно → используйте Gitflow
- TBD для: SaaS (одна версия), микросервисов (независимые версии)
```

**Возражение: "CI слишком медленный"**

Ответ:
```
- Ускорьте CI! Это не возражение, это requirement
- Параллелизм тестов: npm test -- --maxWorkers=8
- Кеширование зависимостей: npm ci --frozen-lockfile
- Отбросить медленные тесты (переписать или удалить)
- Divide на unit (быстрые) и integration (медленные)
- Только unit тесты в PR validation, integration в main
```

## Migration checklist: Gitflow → TBD

```
Инфраструктура:
☐ CI настроен (< 10 минут для полного прохода)
☐ Feature flags система внедрена
☐ Мониторинг на production работает
☐ Alerting настроен (ошибки, latency, crashes)
☐ Rollback процесс документирован и работает

Команда:
☐ Обучение всей команды о TBD
☐ Обучение feature flags API
☐ Обучение мониторингу метрик
☐ Обучение rollback процессу
☐ Определены дежурные (rotation)

Процесс:
☐ Максимальный размер задачи: < 1 дня
☐ Code review процесс чёткий
☐ Merge должен быть < 1 часа (не ждать)
☐ Deploy происходит несколько раз в день
☐ Есть процесс реверта ошибок

Код:
☐ Тестовое покрытие: > 70%
☐ Нет больших monolithic компонентов
☐ Code можно тестировать в isolation
☐ Feature флаги легко добавлять

Метрики:
☐ Среднее время жизни ветки < 1 дня
☐ Коммиты в main: 5-10+ в день на разработчика
☐ Время CI: < 10 минут
☐ % успешных deплоев: > 95%
☐ Среднее время recovery после failed deploy < 15 минут
```

## Часто задаваемые вопросы

**Не опасно ли коммитить незавершённый код в main?** Безопасно при правильном использовании feature flags. Флаг выключен → пользователи не видят незавершённую функцию. CI/CD тестирует что уже написано.

**Подходит ли TBD для маленьких команд?** Зависит от опыта. Опытная маленькая команда — отлично. Начинающие разработчики — лучше начать с GitHub Flow.

**Как делать hotfix в TBD?** Просто коммитим в main (или через быстрый PR). Если нужна отдельная версия — cherry-pick в release branch.

**Когда TBD не подходит?** Open source (разные контрибьюторы), приложения со строгими release циклами (мобильные приложения), команды без хорошего CI/CD.

## Заключение

Trunk-Based Development — разработка с частыми коммитами в main и feature flags для скрытия незавершённого кода. Требует зрелой CI/CD системы и опытной команды. Устраняет проблемы merge hell длинных веток. Используется крупнейшими tech-компаниями. Альтернативы: [GitHub Flow]({{< relref "github-flow" >}}) (менее строгий) и [Gitflow]({{< relref "gitflow" >}}) (для версионных релизов).
