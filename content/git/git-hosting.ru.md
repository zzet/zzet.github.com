---
title: "Git хостинг: сравнение GitHub, GitLab, Bitbucket и Gitea"
description: "Обзор и сравнение платформ для хостинга Git репозиториев: GitHub, GitLab, Bitbucket, Gitea. Цены, возможности, self-hosted варианты."
date: 2026-01-14
lastmod: 2026-01-14
draft: false
slug: "git-hosting"
keywords: ["git хостинг", "хостинг репозиториев git", "github gitlab bitbucket сравнение", "какой git хостинг выбрать", "бесплатный git хостинг", "self-hosted git"]
tags: ["git", "beginner"]
categories: ["git"]
---

Git хостинг — это платформа для хранения репозиториев с веб-интерфейсом, инструментами совместной работы и CI/CD. Выбор платформы влияет на рабочий процесс команды, доступные инструменты и стоимость. Основные игроки: GitHub, GitLab, Bitbucket и self-hosted варианты.

## GitHub

GitHub — крупнейшая платформа для разработчиков с более чем 100 миллионами пользователей. Принадлежит Microsoft.

**Возможности:**
```
✓ Неограниченные публичные и приватные репозитории
✓ GitHub Actions — мощный CI/CD
✓ GitHub Copilot — AI помощник (платно)
✓ GitHub Pages — хостинг статических сайтов
✓ GitHub Packages — npm, Maven, Docker registry
✓ Dependabot — автообновление зависимостей
✓ Code scanning (CodeQL)
✓ Огромное сообщество open source
```

**Тарифы (2024):**
```
Free:    $0/мес   — неограниченные репо, 2000 мин Actions/мес
Team:    $4/польз/мес — 3000 мин Actions, code owners
Enterprise: $21/польз/мес — SAML SSO, advanced security
```

**Идеально для:** open source проектов, JavaScript экосистемы, команд использующих GitHub Actions.

```bash
# Клонирование с GitHub
git clone https://github.com/user/project.git
git clone git@github.com:user/project.git

# GitHub CLI
gh repo clone user/project
gh pr create
gh issue list
```

## GitLab

GitLab — полная DevOps платформа. Основная альтернатива GitHub с мощным CI/CD.

**Возможности:**
```
✓ Встроенный CI/CD (старше GitHub Actions)
✓ Self-hosted (GitLab CE бесплатен)
✓ Container Registry
✓ Package Registry
✓ SAST/DAST безопасность (Ultimate)
✓ Epics и Roadmaps (Premium)
✓ Merge Trains
✓ Terraform state backend
```

**Тарифы:**
```
Free:    $0/мес   — 400 мин CI/CD, 5 ГБ хранилища
Premium: $19/польз/мес — 10 000 мин CI/CD, SAML SSO
Ultimate: $99/польз/мес — 50 000 мин, безопасность, compliance
GitLab CE: бесплатно — self-hosted, open source
```

**Идеально для:** enterprise с требованиями к self-hosted, полного DevOps пайплайна, компаний где важна безопасность.

```bash
# Клонирование с GitLab
git clone https://gitlab.com/user/project.git
git clone git@gitlab.com:user/project.git

# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy
```

## Bitbucket

Bitbucket — платформа от Atlassian. Тесно интегрирована с Jira и Confluence.

**Возможности:**
```
✓ Jira и Confluence интеграция (из коробки)
✓ Bitbucket Pipelines — CI/CD
✓ Pull requests с code review
✓ Branch permissions
✓ Smart Mirroring для enterprise
```

**Тарифы:**
```
Free:    $0/мес   — до 5 пользователей, 50 мин Pipelines/мес
Standard: $3/польз/мес — 2500 мин Pipelines
Premium:  $6/польз/мес — 3500 мин, Merge Checks, SAML
```

**Идеально для:** команд использующих Jira/Confluence для управления задачами.

```bash
# Клонирование с Bitbucket
git clone https://bitbucket.org/user/project.git
git clone git@bitbucket.org:user/project.git
```

## Gitea и Forgejo (Self-Hosted)

Лёгкие self-hosted платформы с минимальными требованиями:

```bash
# Запуск Gitea через Docker
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -p 222:22 \
  -v /srv/gitea:/data \
  gitea/gitea:latest

# Forgejo (fork Gitea)
docker run -d \
  --name forgejo \
  -p 3000:3000 \
  codeberg.org/forgejo/forgejo:latest
```

```
Gitea/Forgejo:
✓ Бесплатный open source
✓ 512 МБ RAM минимум
✓ Полный веб-интерфейс
✓ Pull requests, Issues, CI (Gitea Actions)
✓ GitHub-совместимый API
```

**Идеально для:** небольших команд с собственным сервером, организаций которым нельзя использовать облако.

## Другие платформы

```
SourceHut (sr.ht):
- Минималистичный, email-based workflow
- Платный ($20-$100/год)
- Строгий акцент на открытый исходный код

AWS CodeCommit:
- Интегрирован с AWS экосистемой
- Без лимитов на репозитории
- Хранение в AWS

Azure DevOps:
- Microsoft платформа
- Тесная интеграция с Azure
- Boards, Pipelines, Artifacts

Codeberg.org:
- Некоммерческий, на базе Forgejo
- Бесплатный для open source
- Европейские серверы
```

## Сравнительная таблица

```
Платформа  Free план   CI/CD мин  Self-hosted  Open Source  Особенность
GitHub     2000        2000/мес   Enterprise   Нет          Сообщество
GitLab     400         400/мес    Да (CE бесп) Да (CE)      DevOps
Bitbucket  Да(5 users) 50/мес     Enterprise   Нет          Jira
Gitea      Нет лимитов Локально   Да           Да           Лёгкий
Forgejo    Нет лимитов Локально   Да           Да           Fork Gitea
```

## Миграция между платформами

```bash
# Мигрировать репозиторий (история сохраняется)
git clone --bare https://github.com/user/project.git
cd project.git
git push --mirror https://gitlab.com/user/project.git

# GitHub → GitLab через веб-интерфейс
# GitLab → New project → Import project → GitHub

# GitLab → GitHub через веб-интерфейс
# GitHub → New repository → Import repository

# Bitbucket → GitHub
# GitHub → New repository → Import repository → ввести Bitbucket URL
```

## Как выбрать платформу

```bash
# Открытый проект для сообщества
→ GitHub (крупнейшее сообщество open source)

# Корпоративный проект, нужен self-hosted
→ GitLab CE (бесплатно, полная платформа)

# Команда использует Jira/Atlassian
→ Bitbucket (встроенная интеграция)

# Небольшая команда, свой сервер, минимум ресурсов
→ Gitea / Forgejo

# Нет своего сервера, нужна полная DevOps платформа
→ GitLab.com Premium

# Максимальный CI/CD marketplace и Actions
→ GitHub Team/Enterprise
```

## Часто задаваемые вопросы

**Можно ли использовать несколько платформ одновременно?** Да. Многие проекты зеркалируют репозиторий на нескольких платформах. Или используют GitHub для open source и GitLab/self-hosted для внутренней разработки.

**Какая платформа самая безопасная?** Все крупные платформы имеют сертификацию SOC2, ISO 27001. GitLab предлагает наибольший набор встроенных security инструментов. Self-hosted даёт максимальный контроль.

**Переносятся ли Issues и PR при миграции?** GitLab импортирует Issues и MR с GitHub. GitHub импортирует только код. Для полной миграции Issues используйте специальные инструменты (github-to-gitlab, node-gitlab-migration).

**Стоит ли переходить с GitHub на self-hosted?** Только если есть: требования к хранению данных, ограниченный бюджет на большую команду, или нужен полный контроль. Self-hosted требует администрирования и инфраструктуры.

**Какой хостинг лучше для учебных проектов?** GitHub — наилучший выбор для портфолио. Работодатели привыкли смотреть на GitHub. Публичные репозитории с вашими проектами помогают при найме.

## Заключение

GitHub — лучший выбор для open source и командной разработки с богатым marketplace. GitLab — для enterprise и self-hosted с полным DevOps пайплайном. Bitbucket — для команд в экосистеме Atlassian. Gitea/Forgejo — для лёгкого self-hosted. Подробное сравнение GitHub и GitLab — в статье [GitHub vs GitLab]({{< relref "github-vs-gitlab" >}}).
