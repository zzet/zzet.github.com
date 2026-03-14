---
title: "GitLab: обзор платформы для DevOps и совместной разработки"
description: "Обзор GitLab: репозитории, CI/CD с .gitlab-ci.yml, Issues, Merge Requests, Groups, Runners, Container Registry. Как начать работу с GitLab."
date: 2026-02-07
lastmod: 2026-02-07
draft: false
slug: "gitlab-obzor"
keywords: ["gitlab что это", "gitlab обзор", "как работать с gitlab", "что такое gitlab", "gitlab для чего используется", "gitlab платформа"]
tags: ["git", "gitlab", "beginner"]
categories: ["git"]
---

GitLab — это комплексная DevOps платформа: от хранения кода и code review до CI/CD пайплайнов, управления задачами и мониторинга. В отличие от GitHub, GitLab изначально создавался как полноценная платформа для всего жизненного цикла разработки, а не только для хостинга репозиториев.

## Что такое GitLab

GitLab предоставляет единую платформу для:

```
- Хранение кода (Git репозитории)
- Code review (Merge Requests)
- CI/CD (встроенные пайплайны)
- Управление задачами (Issues, Boards, Milestones)
- Безопасность (SAST, DAST, Dependency Scanning)
- Реестр пакетов и образов (Container Registry, Package Registry)
- Мониторинг и observability
- Документация (Wiki)
```

GitLab доступен как облачный сервис (gitlab.com) и как self-hosted решение (можно развернуть на своём сервере).

## Регистрация и первые шаги

```
1. Открыть gitlab.com
2. Зарегистрироваться (email или OAuth через GitHub/Google)
3. Создать первый проект:
   - New project → Create blank project
   - Указать название, видимость (Public/Internal/Private)
   - Нажать "Create project"
```

После создания GitLab покажет инструкции для подключения локального репозитория:

```bash
# Клонировать существующий проект
git clone git@gitlab.com:username/project.git

# Или подключить локальный репозиторий к GitLab
cd existing-project
git remote add origin git@gitlab.com:username/project.git
git push -u origin main
```

## CI/CD: файл .gitlab-ci.yml

Главное преимущество GitLab — встроенный CI/CD. Для запуска достаточно создать файл `.gitlab-ci.yml` в корне репозитория:

```yaml
# Базовый пайплайн: сборка, тесты, деплой
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building..."
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/

test-job:
  stage: test
  script:
    - npm test
  coverage: '/Coverage: \d+\.\d+%/'

deploy-job:
  stage: deploy
  script:
    - echo "Deploying to production..."
  environment:
    name: production
    url: https://example.com
  only:
    - main
```

GitLab автоматически запустит пайплайн при каждом push. Результаты видны во вкладке CI/CD → Pipelines.

## Структура пайплайна

```yaml
# Глобальные настройки
image: node:18

variables:
  NODE_ENV: test

before_script:
  - npm ci

# Джобы
unit-tests:
  stage: test
  script:
    - npm run test:unit

integration-tests:
  stage: test
  script:
    - npm run test:integration
  allow_failure: true  # пайплайн не упадёт при ошибке

# Условный запуск
deploy-staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  only:
    - develop

deploy-production:
  stage: deploy
  script:
    - ./deploy.sh production
  only:
    - main
  when: manual  # требует ручного запуска
```

## Issues и управление задачами

Issues в GitLab — это задачи, баги и запросы на функции:

```
Создание Issue:
Issues → New issue → Заполнить:
- Title (название)
- Description (описание, поддерживает Markdown)
- Assignee (исполнитель)
- Labels (теги)
- Milestone (веха)
- Due date (срок)
```

**Issue Boards** — Kanban-доски для визуализации работы:

```
Issues → Boards

По умолчанию колонки: Open → In Progress → Done

Можно добавить свои колонки на основе Labels:
- "Backlog" → "In Review" → "Testing" → "Done"
```

## Merge Requests

Merge Request (MR) в GitLab — аналог Pull Request в GitHub. Это предложение влить одну ветку в другую с возможностью code review:

```bash
# Создать ветку и запушить
git checkout -b feature/new-login
# ... сделать изменения ...
git push origin feature/new-login
```

После push GitLab покажет ссылку на создание MR в терминале:

```
remote: To create a merge request for feature/new-login, visit:
remote:   https://gitlab.com/username/project/-/merge_requests/new?merge_request%5Bsource_branch%5D=feature%2Fnew-login
```

Или через интерфейс: Merge Requests → New merge request.

**Настройки MR:**

```
- Approvals — сколько человек должны одобрить перед merge
- Squash commits — объединить коммиты при merge
- Delete source branch — автоматически удалить ветку
- Merge when pipeline succeeds — автоматический merge после CI
```

## Groups и иерархия проектов

GitLab организует проекты через Groups:

```
Структура Groups:
Компания (Group)
├── frontend (Subgroup)
│   ├── web-app (Project)
│   └── mobile-app (Project)
├── backend (Subgroup)
│   ├── api (Project)
│   └── workers (Project)
└── devops (Subgroup)
    └── infrastructure (Project)
```

Преимущества Groups: настройки CI/CD можно применять ко всем проектам группы, управление правами доступа для всей группы сразу.

## GitLab Runners

Runner — это агент, который выполняет джобы CI/CD. GitLab предоставляет shared runners (общие) и позволяет регистрировать свои:

```bash
# Установить GitLab Runner
# Linux:
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Зарегистрировать runner
sudo gitlab-runner register

# Runner спросит:
# URL: https://gitlab.com
# Token: (взять из Settings → CI/CD → Runners)
# Description: my-runner
# Tags: docker, linux
# Executor: docker
# Default image: alpine:latest
```

**Типы runner executors:**

```
Shell    — запускает скрипты в системной оболочке
Docker   — каждая джоба в отдельном Docker контейнере
Kubernetes — запускает джобы в Kubernetes Pod
```

## Container Registry

GitLab включает Container Registry для хранения Docker образов:

```bash
# Авторизоваться
docker login registry.gitlab.com

# Собрать образ
docker build -t registry.gitlab.com/username/project:latest .

# Запушить образ
docker push registry.gitlab.com/username/project:latest

# В .gitlab-ci.yml:
build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

Переменные `$CI_REGISTRY`, `$CI_REGISTRY_USER`, `$CI_REGISTRY_PASSWORD` — предустановленные GitLab переменные.

## Переменные CI/CD и секреты

Секреты (токены, пароли) хранятся как переменные CI/CD:

```
Settings → CI/CD → Variables → Add variable:
- Key: DEPLOY_TOKEN
- Value: secret_value
- Type: Variable или File
- Protected: только для protected branches
- Masked: скрывать в логах
```

Использование в `.gitlab-ci.yml`:

```yaml
deploy:
  script:
    - echo $DEPLOY_TOKEN  # будет masked в логах
    - ./deploy.sh --token $DEPLOY_TOKEN
```

## Environments и деплой

GitLab отслеживает деплои через Environments:

```yaml
deploy-staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com

deploy-production:
  stage: deploy
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://example.com
  when: manual
```

В разделе Deployments → Environments можно видеть историю деплоев для каждого окружения.

## Импорт из GitHub

Если проект уже на GitHub, его можно импортировать в GitLab:

```
1. New project → Import project → GitHub
2. Авторизовать GitLab для доступа к GitHub
3. Выбрать репозитории для импорта
4. GitLab перенесёт: код, issues, pull requests (как MR), вики
```

Или вручную через Git:

```bash
# Клонировать из GitHub
git clone --bare git@github.com:user/project.git

# Пушить зеркало в GitLab
cd project.git
git push --mirror git@gitlab.com:user/project.git
```

## Часто задаваемые вопросы

**Можно ли использовать GitLab бесплатно?** Да. GitLab.com предоставляет бесплатный план с 400 минутами CI/CD в месяц. GitLab Community Edition можно бесплатно развернуть на своём сервере.

**Что такое Auto DevOps?** Auto DevOps — функция GitLab, которая автоматически конфигурирует CI/CD пайплайн без файла `.gitlab-ci.yml`. Определяет язык проекта и создаёт пайплайн автоматически.

**Как отличается GitLab от GitHub?** GitLab — полная DevOps платформа с встроенным CI/CD, управлением задачами, безопасностью. GitHub — больше фокусируется на хостинге кода и экосистеме. Подробнее — [сравнение GitHub и GitLab]({{< relref "github-vs-gitlab" >}}).

**Что такое GitLab Runner и нужен ли он?** Runner выполняет CI/CD задачи. На GitLab.com есть shared runners — они уже настроены. Свой runner нужен для специфических требований или экономии минут CI/CD.

**Как посмотреть логи пайплайна?** CI/CD → Pipelines → выбрать пайплайн → выбрать джобу → видны логи выполнения скриптов.

## Заключение

GitLab — это больше, чем хостинг Git репозиториев. Встроенный CI/CD, управление задачами, Container Registry и средства безопасности делают его полноценной DevOps платформой. Особенно ценен для компаний, которым нужно self-hosted решение или полный контроль над инфраструктурой разработки.
