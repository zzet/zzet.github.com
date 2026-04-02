---
title: "GitLab CE vs EE: отличия Community и Enterprise Edition"
description: "Сравнение GitLab Community Edition (CE) и Enterprise Edition (EE). Что входит в бесплатную версию, что только в EE, как установить, цены."
date: 2026-02-06
lastmod: 2026-04-02
draft: false
slug: "gitlab-ee-vs-ce"
keywords: ["gitlab ee vs ce", "gitlab ce vs ee", "gitlab-ee vs gitlab-ce", "gitlab ee vs gitlab ce", "gitlab community edition", "gitlab enterprise edition отличия", "gitlab ce установка", "gitlab community edition бесплатно", "gitlab ee цена", "отличие gitlab ce от ee"]
tags: ["git", "gitlab", "intermediate"]
categories: ["git"]
---

GitLab распространяется в двух редакциях: Community Edition (CE) — бесплатная с открытым исходным кодом, и Enterprise Edition (EE) — коммерческая с расширенными функциями. Понять разницу важно для выбора при развёртывании на собственном сервере.

## Основная разница

```
GitLab CE (Community Edition):
- Бесплатная, open source (MIT лицензия для большинства компонентов)
- Базовые функции Git, CI/CD, Issues, MR
- Подходит для большинства команд

GitLab EE (Enterprise Edition):
- Платная лицензия
- Включает всё из CE плюс enterprise-функции
- Три уровня: Free, Premium (~$19/пользователь/месяц), Ultimate (~$99/пользователь/месяц)
```

Важный нюанс: на GitLab.com облачный сервис использует EE-кодовую базу. «CE» и «EE» различие актуально прежде всего для self-hosted развёртывания.

## Что включено в GitLab CE (бесплатно)

CE предоставляет полноценные возможности для разработки:

**Репозитории и code review:**

```
✓ Неограниченные публичные и приватные репозитории
✓ Merge Requests с комментариями
✓ Code review (базовый)
✓ Protected branches
✓ Branch permissions
✓ Git LFS
✓ Webhooks
✓ API
```

**CI/CD:**

```
✓ .gitlab-ci.yml пайплайны
✓ Stages, jobs, artifacts
✓ Shared Runners (на gitlab.com, ограниченные минуты)
✓ Self-hosted Runners (без ограничений)
✓ Docker executor
✓ Environments
✓ Auto DevOps (базовый)
```

**Управление задачами:**

```
✓ Issues
✓ Issue Boards (одна доска на проект)
✓ Labels
✓ Milestones
✓ Time tracking
✓ Wiki
✓ Snippets
```

**Безопасность (базовая):**

```
✓ Container Registry
✓ Package Registry (npm, PyPI, Maven и др.)
✓ HTTPS для Git операций
✓ 2FA
```

## Функции только в GitLab EE

**EE Premium (~$19/пользователь/месяц):**

```
Управление командой:
✓ Code Owners — назначение владельцев файлов
✓ Multiple Approvals — несколько обязательных ревьюеров
✓ Group-level boards — доски на уровне группы
✓ Epics — объединение связанных Issues
✓ Roadmaps — планирование на временной шкале

Безопасность и соответствие:
✓ SAML SSO — корпоративная авторизация
✓ SCIM provisioning — автоматизация управления пользователями
✓ Audit events — логирование действий

CI/CD расширенные:
✓ 10 000 минут CI/CD (на gitlab.com)
✓ Merge Trains — последовательные merge с проверкой
✓ Merge Request Approvals — расширенные правила
```

**EE Ultimate (~$99/пользователь/месяц):**

```
Безопасность:
✓ SAST — статический анализ кода
✓ DAST — динамический анализ работающего приложения
✓ Dependency Scanning — анализ зависимостей
✓ Container Scanning — анализ Docker образов
✓ Secret Detection — обнаружение секретов в коде
✓ License Compliance — проверка лицензий

Enterprise-управление:
✓ Geo — географически распределённые зеркала
✓ Portfolio Management
✓ Value Stream Analytics
✓ Advanced compliance
✓ 50 000 минут CI/CD (на gitlab.com)
```

## Уровни доступа (access_level)

В GitLab для обеих редакций используется система ролей с числовыми кодами:

```
10 = Guest     — только просмотр публичного содержимого
20 = Reporter  — просмотр кода, создание Issues
30 = Developer — push в ветки, создание MR
40 = Maintainer — merge MR, управление проектом
50 = Owner     — полный доступ, настройки, удаление
```

Эти коды используются в API и при работе с командной строкой.

## Установка GitLab CE на сервере

```bash
# Ubuntu/Debian
# 1. Установить зависимости
sudo apt-get update
sudo apt-get install -y curl openssh-server postfix

# 2. Добавить репозиторий GitLab CE
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash

# 3. Установить GitLab CE
sudo EXTERNAL_URL="https://gitlab.example.com" apt-get install gitlab-ce

# 4. После установки — получить начальный пароль
sudo cat /etc/gitlab/initial_root_password

# Или с конкретной версией
sudo apt-get install gitlab-ce=16.5.0-ce.0
```

Минимальные требования для сервера: 4 GB RAM, 2 CPU, 50 GB дискового пространства.

## Установка GitLab EE на сервере

```bash
# Аналогично CE, но другой пакет
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash

sudo EXTERNAL_URL="https://gitlab.example.com" apt-get install gitlab-ee

# Для активации лицензии (после установки):
# Admin Area → Subscription → Upload license
```

Лицензия EE — файл `.gitlab-license`, покупается на gitlab.com.

## Конвертация CE в EE

Можно установить CE, а потом активировать EE лицензию:

```bash
# На Debian/Ubuntu
sudo apt-get install gitlab-ee

# GitLab автоматически обновит пакет с CE на EE
# Данные, конфигурация и репозитории сохранятся

# Убедиться в успехе
sudo gitlab-rake gitlab:check
```

Обратная конвертация (EE → CE) также возможна, но enterprise-функции будут недоступны.

## Основные команды управления self-hosted GitLab

```bash
# Перезапустить GitLab
sudo gitlab-ctl restart

# Проверить статус сервисов
sudo gitlab-ctl status

# Посмотреть логи
sudo gitlab-ctl tail

# Выполнить backup
sudo gitlab-backup create

# Восстановить из backup
sudo gitlab-backup restore BACKUP=timestamp_of_backup

# Изменить конфигурацию
sudo nano /etc/gitlab/gitlab.rb
sudo gitlab-ctl reconfigure  # применить изменения

# Проверить версию
sudo gitlab-rake gitlab:env:info
```

## Сравнение self-hosted vs gitlab.com

```
Self-hosted GitLab CE:
+ Бесплатно
+ Полный контроль над данными
+ Неограниченные минуты CI/CD (на своих раннерах)
- Требует сервера и администрирования
- Нужно самостоятельно обновлять
- Настройка резервного копирования на вас

GitLab.com (Free plan):
+ Не нужно своего сервера
+ Обслуживает GitLab Inc.
+ Автоматические обновления
- 400 минут CI/CD в месяц
- Данные хранятся у GitLab
```

## Часто задаваемые вопросы

**Чем отличается GitLab CE от бесплатного плана на gitlab.com?** CE — программное обеспечение для self-hosted. GitLab.com Free — облачный сервис. Кодовая база на gitlab.com использует EE, но большинство функций EE требуют платной лицензии.

**Нужен ли EE для коммерческого использования?** Нет. GitLab CE можно использовать в коммерческих проектах бесплатно. EE нужен только для enterprise-функций (SAML SSO, Geo, расширенная безопасность и т. д.).

**Можно ли начать с CE и потом перейти на EE?** Да, это стандартный путь. Установите CE, и когда понадобятся EE функции — купите лицензию и активируйте. Данные сохранятся.

**Что такое GitLab Omnibus?** Это метод установки GitLab, который упаковывает все компоненты (PostgreSQL, Redis, Nginx, GitLab) в один deb/rpm пакет. Стандартный способ установки для большинства.

**Как узнать, какая версия установлена?** `sudo gitlab-rake gitlab:env:info` покажет версию и тип (CE или EE). В веб-интерфейсе: Admin Area → Dashboard.

## Заключение

GitLab CE — мощная бесплатная платформа, которая закрывает потребности большинства команд. EE Premium добавляет корпоративные функции управления: Code Owners, SAML SSO, Epics. EE Ultimate — для enterprise с требованиями безопасности: SAST, DAST, сканирование зависимостей. Для большинства небольших и средних команд GitLab CE — оптимальный выбор. Подробнее о ролях в GitLab — [система ролей GitLab]({{< relref "gitlab-reporter" >}}).
