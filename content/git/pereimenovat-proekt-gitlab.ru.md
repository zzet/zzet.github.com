---
title: "Как переименовать проект в GitLab: имя и путь репозитория"
description: "Пошаговая инструкция по переименованию проекта в GitLab. Разница между именем и путём проекта, обновление git remote после смены URL."
date: 2026-02-23
lastmod: 2026-02-23
draft: false
slug: "pereimenovat-proekt-gitlab"
keywords: ["переименовать проект gitlab", "изменить название репозитория gitlab", "gitlab rename project", "gitlab rename repository", "изменить url репозитория gitlab", "gitlab project rename"]
tags: ["git", "gitlab", "beginner"]
categories: ["git"]
---

В GitLab у проекта есть два разных параметра: отображаемое имя (название) и путь (slug в URL). Их можно изменять независимо. Понять разницу важно, потому что изменение пути меняет URL репозитория и требует обновления `git remote` на всех клонах.

## Разница между именем и путём проекта

```
Имя проекта (Name):
- Отображается в заголовке страницы и списках
- Не влияет на URL
- Может содержать пробелы и специальные символы
- Пример: "My Awesome Project"

Путь проекта (Path):
- Используется в URL репозитория
- Влияет на адрес для git clone
- Только латинские буквы, цифры, дефис, подчёркивание
- Пример: my-awesome-project
  URL: gitlab.com/username/my-awesome-project
```

## Изменить отображаемое имя проекта

Смена имени (без изменения URL):

```
1. Открыть проект в GitLab
2. Settings → General
3. В разделе "Naming, topics, avatar":
   - Поле "Project name" — изменить на новое имя
4. Нажать "Save changes"
```

URL остаётся прежним. Все клоны и remote продолжают работать без изменений.

## Изменить путь проекта (URL меняется)

Смена пути — более серьёзная операция:

```
1. Открыть проект в GitLab
2. Settings → General
3. Раскрыть раздел "Advanced"
4. Найти "Change path"
5. Ввести новый путь (slug)
   Например: old-name → new-name
6. Нажать "Change path"
7. GitLab попросит подтвердить, введя текущий путь
```

После смены пути:
- Старый URL перестаёт работать
- GitLab создаёт временный редирект со старого URL (некоторое время)
- Все участники команды должны обновить `git remote`

## Обновить git remote после смены пути

После изменения пути проекта в GitLab URL репозитория изменился. Нужно обновить remote во всех локальных клонах:

```bash
# Посмотреть текущий remote
git remote -v
# origin  git@gitlab.com:username/old-name.git (fetch)
# origin  git@gitlab.com:username/old-name.git (push)

# Обновить remote URL
git remote set-url origin git@gitlab.com:username/new-name.git

# Проверить
git remote -v
# origin  git@gitlab.com:username/new-name.git (fetch)
# origin  git@gitlab.com:username/new-name.git (push)

# Убедиться что всё работает
git fetch origin
```

Для HTTPS URL:

```bash
git remote set-url origin https://gitlab.com/username/new-name.git
```

## GitLab сохраняет редирект

GitLab автоматически создаёт редирект со старого URL на новый. Это значит:

```
Поведение GitLab после смены пути:
- git clone по старому URL → работает (через редирект)
- git fetch / git push → работает некоторое время
- Если кто-то займёт старый путь — редирект пропадёт
```

Несмотря на редиректы, рекомендуется сразу обновить `git remote` во всех клонах — полагаться на редиректы не стоит.

## Переместить проект в другую группу

Связанная операция — перенос проекта в другой Namespace (из личного в группу или между группами):

```
1. Settings → General → Advanced
2. Раздел "Transfer project"
3. Выбрать новый Namespace из выпадающего списка
4. Ввести имя проекта для подтверждения
5. Нажать "Transfer project"
```

После переноса URL изменится:

```bash
# Было:
# git@gitlab.com:username/project.git

# Стало (перенесли в группу "mycompany"):
# git@gitlab.com:mycompany/project.git

# Обновить remote
git remote set-url origin git@gitlab.com:mycompany/project.git
```

## Переименование через GitLab API

Можно переименовать проект программно через API:

```bash
# Изменить имя проекта (name)
curl --request PUT \
  --header "PRIVATE-TOKEN: <your_token>" \
  --data "name=New Project Name" \
  "https://gitlab.com/api/v4/projects/<project_id>"

# Изменить путь проекта (path/URL)
curl --request PUT \
  --header "PRIVATE-TOKEN: <your_token>" \
  --data "path=new-project-path" \
  "https://gitlab.com/api/v4/projects/<project_id>"

# Изменить и имя и путь одновременно
curl --request PUT \
  --header "PRIVATE-TOKEN: <your_token>" \
  --data "name=New Name&path=new-path" \
  "https://gitlab.com/api/v4/projects/<project_id>"

# Узнать ID проекта
curl --header "PRIVATE-TOKEN: <your_token>" \
  "https://gitlab.com/api/v4/projects?search=project-name" | \
  python3 -m json.tool | grep '"id"'
```

## Что нужно обновить после переименования

Если путь проекта изменился, проверьте все места, где был указан старый URL:

```
Что нужно обновить:
□ git remote в локальных клонах всех разработчиков
□ CI/CD конфигурации (если URL захардкожен)
□ Документация с примерами git clone
□ README.md с бейджами и ссылками
□ Webhooks (старый URL перестанет работать)
□ Деплой скрипты с URL репозитория
□ .gitlab-ci.yml если есть ссылки на репозиторий
□ Зависимости через git (package.json, Gemfile и т.д.)
```

## Практические примеры

```bash
# Проверить текущие remote перед обновлением
git remote -v

# Обновить SSH remote
git remote set-url origin git@gitlab.com:mygroup/new-project-name.git

# Обновить HTTPS remote
git remote set-url origin https://gitlab.com/mygroup/new-project-name.git

# Проверить, что remote обновился корректно
git fetch origin
git status

# Если несколько remote (например, есть upstream)
git remote -v
git remote set-url upstream git@gitlab.com:upstream-org/new-project-name.git

# Скрипт для массового обновления (для всех клонов в папке)
# Найти все репозитории с устаревшим URL
grep -r "old-name.git" */.git/config
```

## Часто задаваемые вопросы

**Нужны ли особые права для переименования?** Да, роль Maintainer или Owner. Developer не может изменять настройки проекта.

**Сломаются ли существующие клоны после смены пути?** GitLab создаёт редиректы, поэтому не сразу. Но рекомендуется обновить `git remote set-url` как можно скорее — редиректы могут пропасть, если кто-то займёт старый путь.

**Можно ли вернуть старый путь?** Да, если он не занят другим проектом. Перейдите в Settings → Advanced → Change path и введите прежнее значение.

**Меняется ли путь к Container Registry при переименовании?** Да. Если в `.gitlab-ci.yml` или скриптах захардкожен путь к registry (`registry.gitlab.com/username/old-name`), его нужно обновить.

**Что происходит с открытыми Issues и MR при переименовании?** Они остаются с новым URL. GitLab сохраняет все данные — Issues, MR, комментарии, теги.

## Заключение

Переименование проекта в GitLab: Settings → General для смены отображаемого имени (без изменения URL), Settings → General → Advanced → Change path для смены URL. После смены пути обязательно обновите `git remote set-url origin <new-url>` в локальных клонах. GitLab создаёт временные редиректы, но полагаться на них не стоит. Для управления проектом также нужно знать [как удалить проект в GitLab]({{< relref "udalit-proekt-gitlab" >}}).
