---
title: "Как удалить проект в GitLab: пошаговая инструкция"
description: "Инструкция по удалению репозитория GitLab. Настройки безопасного удаления, резервная копия перед удалением, восстановление удалённого проекта, API."
date: 2026-03-11
lastmod: 2026-03-11
draft: false
slug: "udalit-proekt-gitlab"
keywords: ["удалить проект gitlab", "как удалить репозиторий gitlab", "gitlab delete project", "gitlab удалить репозиторий", "gitlab delete repository", "как удалить проект gitlab"]
tags: ["git", "gitlab", "beginner"]
categories: ["git"]
---

Удалить проект в GitLab можно через Settings или API. Важно понимать последствия: на GitLab.com есть период отложенного удаления (7 дней), в течение которого проект можно восстановить. При self-hosted GitLab настройки могут отличаться.

## Требования для удаления

Удалить проект может только владелец (Owner). Maintainer не имеет права удалять проекты:

```
Необходимые права: Owner (access_level = 50)

Если вы не Owner:
- Settings → Members → проверить свою роль
- Попросить Owner удалить проект или повысить роль
```

## Создать резервную копию перед удалением

Рекомендуется сохранить копию репозитория перед удалением:

```bash
# Клонировать репозиторий со всеми ветками и тегами
git clone --bare git@gitlab.com:username/project.git

# bare-клон сохраняет всю историю, ветки и теги
# Папка project.git содержит полный репозиторий

# Для будущего восстановления:
cd project.git
git bundle create ../project-backup.bundle --all

# Восстановить из bundle:
git clone project-backup.bundle restored-project
```

## Удаление проекта через веб-интерфейс

```
1. Открыть проект на GitLab

2. Перейти в Settings → General

3. Раскрыть раздел "Advanced"

4. Прокрутить вниз до "Delete project"

5. Нажать кнопку "Delete project"

6. В диалоге ввести:
   - Имя проекта (точно как оно указано)
   - Пароль аккаунта GitLab

7. Нажать "Yes, delete project"
```

На GitLab.com проект не удаляется мгновенно — включается период отложенного удаления (7 дней).

## Отложенное удаление на GitLab.com

GitLab.com использует политику отложенного удаления (Delayed Project Deletion):

```
После нажатия "Delete project":
- Проект помечается как "pending deletion"
- Он недоступен для новых операций
- Через 7 дней удаляется окончательно

В течение 7 дней можно восстановить:
1. Manage → Projects → Deleted projects
   (или Admin Area → Projects → Deleted projects)
2. Найти проект в списке
3. Нажать "Restore"
```

После 7 дней — восстановление невозможно без резервной копии.

## Удаление проекта через GitLab API

```bash
# Узнать ID проекта
curl --header "PRIVATE-TOKEN: <your_token>" \
  "https://gitlab.com/api/v4/projects?search=project-name" | \
  python3 -m json.tool | grep '"id"'

# Удалить проект по ID
curl --request DELETE \
  --header "PRIVATE-TOKEN: <your_token>" \
  "https://gitlab.com/api/v4/projects/<project_id>"

# Удалить проект по namespace/path
curl --request DELETE \
  --header "PRIVATE-TOKEN: <your_token>" \
  "https://gitlab.com/api/v4/projects/username%2Fproject-name"
# (слэш в пути кодируется как %2F)
```

API-удаление также подпадает под политику отложенного удаления.

## Восстановление удалённого проекта

Если проект был удалён и прошло не более 7 дней (GitLab.com):

```
Способ 1: через интерфейс
1. Меню слева → Explore → Projects
   или Admin Area → Projects (для администраторов)
2. Фильтр "Deleted"
3. Найти нужный проект
4. Нажать "Restore"

Способ 2: через API
curl --request POST \
  --header "PRIVATE-TOKEN: <your_token>" \
  "https://gitlab.com/api/v4/projects/<project_id>/restore"
```

## Отключить локальный репозиторий от удалённого

После удаления проекта на GitLab, если у вас остался локальный клон:

```bash
# Проверить текущие remote
git remote -v
# origin  git@gitlab.com:username/deleted-project.git (fetch)

# Убрать ссылку на удалённый репозиторий
git remote remove origin

# Теперь локальная история сохранена, но нет remote
git remote -v
# (пустой вывод)

# Можно подключить к новому репозиторию
git remote add origin git@gitlab.com:username/new-project.git
git push -u origin main
```

## Удаление группы проектов

Если нужно удалить группу (и все проекты в ней):

```
1. Открыть группу
2. Settings → General
3. Раздел "Advanced"
4. "Remove group"
5. Ввести имя группы для подтверждения

Внимание: удаляются ВСЕ проекты в группе и подгруппы
```

Перед удалением группы проверьте список проектов: Group → Projects.

## Мягкое удаление (архивирование) как альтернатива

Если проект больше не нужен для работы, но нужно сохранить историю:

```
Settings → General → Advanced → Archive project

Результат:
- Проект переходит в read-only режим
- Не появляется в стандартных поисках
- История и код сохранены
- Можно разархивировать в любой момент
```

Архивирование — более безопасная альтернатива удалению для случаев, когда проект может понадобиться в будущем.

## Практические примеры

```bash
# Сохранить backup перед удалением
git clone --bare git@gitlab.com:username/project.git project-backup.git
tar -czf project-backup.tar.gz project-backup.git

# Проверить backup
git -C project-backup.git log --oneline | head -20

# Удалить через API с токеном
PROJECT_ID=12345
TOKEN="your_private_token"

curl --request DELETE \
  --header "PRIVATE-TOKEN: $TOKEN" \
  "https://gitlab.com/api/v4/projects/$PROJECT_ID"

# Проверить статус удаления
curl --header "PRIVATE-TOKEN: $TOKEN" \
  "https://gitlab.com/api/v4/projects/$PROJECT_ID" | \
  python3 -m json.tool | grep -E '"id"|"name"|"marked_for_deletion"'

# Восстановить удалённый проект через API
curl --request POST \
  --header "PRIVATE-TOKEN: $TOKEN" \
  "https://gitlab.com/api/v4/projects/$PROJECT_ID/restore"
```

## Часто задаваемые вопросы

**Можно ли восстановить проект после удаления?** На GitLab.com — да, в течение 7 дней через раздел удалённых проектов. После 7 дней — только из резервной копии. При self-hosted GitLab настройки периода удаления может изменить администратор.

**Удаляются ли Issues и MR при удалении проекта?** Да, всё удаляется: репозиторий, история, Issues, Merge Requests, вики, Container Registry.

**Что происходит с форками при удалении исходного проекта?** Форки продолжают работать независимо. GitLab не удаляет форки при удалении исходного проекта.

**Как удалить один из нескольких проектов в группе, не затрагивая другие?** Удаляйте через Settings проекта (не группы). Удаление группы удаляет всё внутри.

**Нужно ли как-то закрыть Issues перед удалением?** Нет, это необязательно. Issues, MR и все данные удалятся вместе с проектом автоматически.

**Можно ли удалить проект если он форкнут другими?** Да. Удаление исходного проекта не удаляет форки. Форки становятся независимыми репозиториями.

## Заключение

Удалить проект в GitLab: Settings → General → Advanced → Delete project, ввести имя и пароль. На GitLab.com проект можно восстановить в течение 7 дней. Перед удалением создайте backup через `git clone --bare`. Для временного скрытия без удаления — используйте архивирование. Как переименовать проект вместо удаления — [переименование проекта в GitLab]({{< relref "pereimenovat-proekt-gitlab" >}}).
