---
title: "Gitflow: модель ветвления для управляемых релизов"
description: "Полное руководство по Gitflow workflow. Ветки main, develop, feature, release, hotfix. Установка git-flow, команды, когда использовать Gitflow."
date: 2026-02-03
lastmod: 2026-02-03
draft: false
slug: "gitflow"
keywords: ["gitflow", "git flow workflow", "gitflow ветки", "gitflow workflow", "gitflow develop ветка", "gitflow release branch", "как работает gitflow"]
tags: ["git", "workflow", "intermediate"]
categories: ["git"]
---

Gitflow — это модель ветвления Git, предложенная Винсентом Дриссеном в 2010 году. Определяет строгую структуру веток и правила их использования. Популярна в проектах с плановыми релизами и поддержкой нескольких версий.

## Структура веток Gitflow

```
main (production)
  └── всегда стабильный код
  └── каждый коммит — тег версии

develop (интеграция)
  └── основная ветка разработки
  └── объединяет feature ветки

feature/* (функции)
  └── создаются от develop
  └── сливаются обратно в develop

release/* (подготовка релиза)
  └── создаются от develop
  └── сливаются в main И develop

hotfix/* (срочные исправления)
  └── создаются от main
  └── сливаются в main И develop
```

## Установка git-flow

```bash
# macOS (Homebrew)
brew install git-flow

# Ubuntu/Debian
sudo apt-get install git-flow

# Windows (Git for Windows включает git-flow)
# Или: https://github.com/petervanderdoes/gitflow-avh

# Инициализировать Gitflow в репозитории
git flow init

# Вопросы при init (принять значения по умолчанию или настроить):
# Branch name for production releases: [main]
# Branch name for "next release" development: [develop]
# Feature branches prefix: [feature/]
# Bugfix branches prefix: [bugfix/]
# Release branches prefix: [release/]
# Hotfix branches prefix: [hotfix/]
```

## Feature ветки

```bash
# Начать разработку функции
git flow feature start user-authentication
# Создаёт ветку feature/user-authentication от develop

# Разработка...
git commit -am "feat: implement login form"
git commit -am "feat: add JWT token handling"
git commit -am "test: add auth service tests"

# Завершить feature (merge в develop)
git flow feature finish user-authentication
# Сливает в develop, удаляет feature/user-authentication

# Альтернатива через обычный Git:
git switch -c feature/user-authentication develop
# ... разработка ...
git switch develop
git merge --no-ff feature/user-authentication
git branch -d feature/user-authentication
```

## Release ветки

Release ветка используется для финальной подготовки релиза:

```bash
# Начать подготовку релиза 1.2.0
git flow release start 1.2.0
# Создаёт release/1.2.0 от develop

# Финальные правки:
# - Обновить версию в package.json
# - Обновить CHANGELOG
# - Исправить мелкие баги

git commit -am "chore: bump version to 1.2.0"
git commit -am "docs: update CHANGELOG"

# Завершить релиз (merge в main И develop)
git flow release finish 1.2.0
# 1. Сливает release/1.2.0 в main
# 2. Создаёт тег v1.2.0
# 3. Сливает release/1.2.0 в develop
# 4. Удаляет release/1.2.0

# Отправить изменения
git push origin main develop --tags
```

## Hotfix ветки

Срочные исправления продакшн багов:

```bash
# Начать hotfix для версии 1.2.1
git flow hotfix start 1.2.1
# Создаёт hotfix/1.2.1 от main

# Исправить критический баг
git commit -am "fix: patch SQL injection vulnerability"

# Завершить hotfix (merge в main И develop)
git flow hotfix finish 1.2.1
# 1. Сливает hotfix/1.2.1 в main
# 2. Создаёт тег v1.2.1
# 3. Сливает hotfix/1.2.1 в develop
# 4. Удаляет hotfix/1.2.1

git push origin main develop --tags
```

## Полный жизненный цикл релиза

```
1. От develop создаются feature ветки
   git flow feature start new-feature

2. Feature ветки сливаются обратно в develop
   git flow feature finish new-feature

3. Когда develop готов к релизу — release ветка
   git flow release start 2.0.0

4. Тестирование и финальные правки в release ветке
   git commit -am "fix: minor bugs found in QA"

5. Завершение релиза — merge в main и develop, тег
   git flow release finish 2.0.0

6. Если нашли баг в production — hotfix
   git flow hotfix start 2.0.1
   git flow hotfix finish 2.0.1
```

## Когда использовать Gitflow

Gitflow хорошо подходит для:

```
✓ Проекты с плановыми релизами (каждые 2-4 недели)
✓ Несколько версий в production одновременно
✓ Большие команды с QA процессом
✓ Open source проекты с maintained releases
✓ Мобильные приложения (App Store/Play Store цикл)
```

Gitflow плохо подходит для:

```
✗ Continuous deployment (деплой несколько раз в день)
✗ Маленькие команды (overhead выше пользы)
✗ SaaS проекты с одной версией в production
✗ Startup стадия (нужна скорость, не процесс)
```

## Альтернативы Gitflow

```
GitHub Flow: проще — только main + feature ветки
  Для: команды с CD, web-сервисы

Trunk-Based Development: все в main, Feature flags
  Для: опытные команды с хорошим CI/CD

GitLab Flow: GitFlow + Environment branches
  Для: несколько окружений (staging, production)
```

## Частые ошибки при Gitflow

```bash
# ОШИБКА: создать feature от main вместо develop
git switch -c feature/wrong main  # неправильно
git flow feature start correct    # правильно (от develop)

# ОШИБКА: слить feature напрямую в main
git merge feature/new main  # неправильно
git flow feature finish new # правильно (через develop)

# ОШИБКА: не создавать release ветку для крупных релизов
# — нет места для QA и финальных правок
```

## Часто задаваемые вопросы

**Обязательно ли использовать инструмент git-flow?** Нет. Gitflow — это модель, не инструмент. Можно следовать ей через обычные git команды. Инструмент `git flow` лишь автоматизирует стандартные шаги.

**Почему hotfix идёт в develop, а не только в main?** Исправление бага должно быть в обоих местах: в main (производство) и в develop (следующий релиз). Иначе баг вернётся в следующем релизе.

**Можно ли иметь несколько release веток одновременно?** Не рекомендуется — усложняет управление. Обычно одна release ветка — один готовящийся релиз.

**Что делать если hotfix конфликтует с develop?** Разрешить конфликты при финише hotfix. Это нормально — develop ушёл вперёд пока шла разработка.

## Заключение

Gitflow определяет структуру веток: `main` (production), `develop` (интеграция), `feature/*` (функции), `release/*` (подготовка), `hotfix/*` (срочные фиксы). Подходит для проектов с плановыми релизами. Для непрерывного деплоя — рассмотрите [GitHub Flow]({{< relref "github-flow" >}}) или [Trunk-Based Development]({{< relref "trunk-based-development" >}}).
