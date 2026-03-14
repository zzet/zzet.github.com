---
title: "Роли в GitLab: Guest, Reporter, Developer, Maintainer, Owner"
description: "Полное руководство по системе ролей GitLab. Что может Reporter, Developer и Maintainer. Как назначить роль через интерфейс и API."
date: 2026-02-07
lastmod: 2026-02-07
draft: false
slug: "gitlab-reporter"
keywords: ["gitlab reporter роль", "роли gitlab", "gitlab права доступа", "gitlab developer роль", "gitlab maintainer права", "доступ к репозиторию gitlab"]
tags: ["git", "gitlab", "intermediate"]
categories: ["git"]
---

GitLab использует систему ролей для управления правами доступа к проектам и группам. Каждая роль определяет, что участник может делать: просматривать код, создавать задачи, делать коммиты, управлять проектом. Понимание ролей важно для правильной организации работы команды.

## Пять ролей GitLab

GitLab определяет пять уровней доступа:

```
Guest (10)       — минимальный доступ
Reporter (20)    — просмотр и задачи
Developer (30)   — разработка
Maintainer (40)  — управление проектом
Owner (50)       — полный контроль
```

Числа в скобках — это коды уровней доступа (`access_level`) для API.

## Guest (уровень 10)

Guest — самый ограниченный уровень. Подходит для внешних пользователей, которым нужен минимальный доступ:

```
Может:
✓ Просматривать публичные Issues и комментарии
✓ Создавать Issues (в публичных и внутренних проектах)
✓ Просматривать список веток и тегов
✓ Скачивать код (если проект не private)

Не может:
✗ Просматривать код в private-репозиториях
✗ Клонировать private-репозиторий
✗ Создавать Merge Requests
✗ Пушить код
✗ Просматривать CI/CD пайплайны
```

## Reporter (уровень 20)

Reporter видит код и создаёт задачи, но не может вносить изменения в репозиторий:

```
Может (всё что Guest +):
✓ Клонировать и скачивать репозиторий
✓ Просматривать весь код
✓ Создавать и редактировать Issues
✓ Комментировать Merge Requests
✓ Просматривать CI/CD пайплайны и их логи
✓ Просматривать Container Registry
✓ Просматривать вики

Не может:
✗ Пушить код (ни в какие ветки)
✗ Создавать ветки
✗ Создавать Merge Requests
✗ Запускать CI/CD пайплайны
✗ Управлять настройками проекта
```

Reporter подходит для: QA-инженеров, которые только создают баги; стейкхолдеров, которые следят за прогрессом; аналитиков, которым нужно читать код.

## Developer (уровень 30)

Developer — основная роль для разработчиков в команде:

```
Может (всё что Reporter +):
✓ Клонировать и пушить код
✓ Создавать и удалять ветки (кроме protected)
✓ Создавать теги (кроме protected)
✓ Создавать и редактировать Merge Requests
✓ Назначать MR ревьюерам
✓ Запускать, отменять и повторять CI/CD пайплайны
✓ Создавать и удалять Environments
✓ Управлять своими Snippets

Не может:
✗ Пушить в protected branches напрямую (по умолчанию)
✗ Принимать (merge) Merge Requests (по умолчанию)
✗ Управлять Runners
✗ Изменять настройки проекта
✗ Приглашать участников
```

## Maintainer (уровень 40)

Maintainer управляет проектом и имеет почти полный контроль:

```
Может (всё что Developer +):
✓ Пушить в protected branches
✓ Принимать Merge Requests
✓ Управлять ветками (создавать, удалять, защищать)
✓ Управлять тегами
✓ Добавлять/удалять участников (до уровня Developer)
✓ Настраивать CI/CD переменные
✓ Управлять Runners
✓ Управлять Container Registry
✓ Настраивать Webhooks
✓ Редактировать описание проекта и аватар
✓ Переименовать проект

Не может:
✗ Изменить видимость проекта (Public/Private)
✗ Удалить проект
✗ Добавлять пользователей с ролью Owner
✗ Изменить Namespace проекта
```

## Owner (уровень 50)

Owner — полный контроль над проектом:

```
Может всё что Maintainer +:
✓ Изменить видимость проекта
✓ Удалить проект
✓ Архивировать проект
✓ Настроить форки
✓ Передать проект другому Namespace
✓ Управлять всеми интеграциями
✓ Добавлять пользователей с любой ролью
```

В личных проектах (не в группе) создатель автоматически становится Owner.

## Как назначить роль через веб-интерфейс

```
Для проекта:
1. Открыть проект
2. Manage → Members (или Settings → Members)
3. Нажать "Invite members"
4. Ввести username или email
5. Выбрать роль из выпадающего списка
6. Опционально: установить срок истечения (Expiration date)
7. Нажать "Invite"

Для группы:
1. Открыть группу
2. Manage → Members
3. "Invite members"
4. Роль в группе наследуется всеми проектами группы
```

## Управление ролями через API

GitLab API позволяет управлять ролями программно:

```bash
# Получить список участников проекта
curl --header "PRIVATE-TOKEN: <your_token>" \
  "https://gitlab.com/api/v4/projects/<project_id>/members"

# Добавить пользователя как Reporter (access_level=20)
curl --request POST \
  --header "PRIVATE-TOKEN: <your_token>" \
  --data "user_id=5&access_level=20" \
  "https://gitlab.com/api/v4/projects/<project_id>/members"

# Изменить роль пользователя на Developer (access_level=30)
curl --request PUT \
  --header "PRIVATE-TOKEN: <your_token>" \
  --data "access_level=30" \
  "https://gitlab.com/api/v4/projects/<project_id>/members/5"

# Удалить пользователя из проекта
curl --request DELETE \
  --header "PRIVATE-TOKEN: <your_token>" \
  "https://gitlab.com/api/v4/projects/<project_id>/members/5"
```

Числовые значения `access_level`: Guest=10, Reporter=20, Developer=30, Maintainer=40, Owner=50.

## Коды доступа в конфигурации

При настройке protected branches в `.gitlab-ci.yml` или через API также используются эти коды:

```yaml
# Пример защиты ветки через API
# Разрешить push только Maintainer и выше
curl --request POST \
  --header "PRIVATE-TOKEN: <token>" \
  --data "name=main&push_access_level=40&merge_access_level=40" \
  "https://gitlab.com/api/v4/projects/:id/protected_branches"
```

```
push_access_level / merge_access_level:
0  = No access (никто)
30 = Developer and above (Developer, Maintainer, Owner)
40 = Maintainer and above (Maintainer, Owner)
60 = Admin only
```

## Protected Branches и роли

По умолчанию ветка `main` (или `master`) protected. Это означает:

```
Protected branch (настройки по умолчанию):
- Push: разрешён только Maintainer и выше
- Merge: разрешён только Maintainer и выше
- Developer: может создавать MR, но не может напрямую пушить

Настройка через:
Settings → Repository → Protected branches
```

Можно настроить так, чтобы Developer тоже мог пушить в определённые ветки.

## Роли в Group vs Project

Роль назначается на уровне группы или проекта:

```
Группа определяет максимальный уровень доступа:
- Если в группе: Developer
- В проекте можно повысить до Maintainer
- Но нельзя понизить ниже уровня группы

Пример:
Group "backend" → user Alex = Developer
  Project "api" → Alex = Maintainer (повышение разрешено)
  Project "legacy" → Alex = Reporter (понижение запрещено, будет Developer)
```

## Временный доступ

GitLab позволяет назначать роль с истечением срока:

```
При добавлении участника:
- Expiration date: выбрать дату

После этой даты пользователь автоматически теряет доступ.
Полезно для: подрядчиков, стажёров, временных коллабораторов.
```

## Часто задаваемые вопросы

**Может ли Reporter делать fork репозитория?** Да, если это разрешено настройками проекта (Forking allowed). По умолчанию — да.

**Как дать Developer возможность merge MR?** В настройках защищённой ветки: Settings → Repository → Protected branches → изменить "Allowed to merge" на "Developers + Maintainers".

**Отличаются ли роли для GitLab CE и EE?** Базовые роли (Guest/Reporter/Developer/Maintainer/Owner) одинаковые. EE добавляет функции типа Code Owners, которые влияют на то, кто должен одобрить MR, но сами роли неизменны.

**Можно ли дать пользователю разные роли в разных проектах?** Да. Назначьте роль на уровне конкретного проекта. Она будет применяться только к этому проекту, независимо от других.

**Что значит access_level в логах и API?** Числовой код роли: 10=Guest, 20=Reporter, 30=Developer, 40=Maintainer, 50=Owner. Используется в GitLab API при управлении участниками программно.

## Заключение

Система ролей GitLab — Guest (10), Reporter (20), Developer (30), Maintainer (40), Owner (50) — покрывает все сценарии управления доступом. Reporter видит код и создаёт задачи, но не пушит. Developer пишет код и создаёт MR. Maintainer принимает MR и управляет проектом. Для понимания разницы в редакциях GitLab — [GitLab CE vs EE]({{< relref "gitlab-ee-vs-ce" >}}).
