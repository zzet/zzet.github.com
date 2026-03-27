---
title: "git log: полное руководство по просмотру истории коммитов"
description: "Научитесь использовать git log для просмотра истории коммитов. Фильтры, форматирование, примеры команд для анализа истории проекта."
date: 2026-01-19
lastmod: 2026-01-19
draft: false
slug: "git-log"
keywords: ["посмотреть коммиты в ветке git", "git log", "история коммитов", "git log что делает", "git log --oneline", "git log format", "git log --all --graph", "посмотреть историю коммитов git"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Команда `git log` — главный инструмент для изучения истории проекта. Она показывает, что, кто и когда изменил, помогает найти причину ошибки, понять решения прошлого. При умелом использовании `git log` — это мощный инструмент анализа.

## Базовый синтаксис

```bash
git log
```

Выводит список коммитов от новых к старым с полной информацией:

```
commit a1b2c3d4e5f6789012345678901234567890abcd
Author: Иван Петров <ivan@example.com>
Date:   Thu Mar 13 15:30:00 2026 +0300

    feat: добавлена OAuth2 авторизация

    Интеграция с Google OAuth2.
    Добавлены эндпоинты /auth/google и /auth/callback.

    Closes #123
```

Нажмите `q` для выхода из просмотра. `j`/`k` или стрелки для прокрутки.

## Форматирование вывода

**Компактный формат (самый используемый):**

```bash
git log --oneline
# a1b2c3d feat: добавлена OAuth2 авторизация
# b2c3d4e fix: исправлена ошибка в корзине
# c3d4e5f docs: обновлён README
```

**С графом веток:**

```bash
git log --graph --oneline --all
# * a1b2c3d (HEAD -> main) feat: добавлена OAuth2
# | * b2c3d4e (feature/cart) fix: исправлена корзина
# |/
# * c3d4e5f docs: обновлён README
```

**Показать все ветки:**

```bash
git log --all --oneline
```

**Показать декораторы (имена веток и тегов):**

```bash
git log --decorate --oneline
# a1b2c3d (HEAD -> main, origin/main) feat: OAuth2
```

**Кастомный формат:**

```bash
# %h — короткий хеш, %an — автор, %ar — когда, %s — тема
git log --pretty=format:"%h - %an, %ar : %s"
# a1b2c3d - Иван Петров, 2 часа назад : feat: OAuth2

# Ещё варианты:
# %H — полный хеш
# %ae — email автора
# %ad — дата в формате автора
# %cd — дата коммита
# %cr — дата относительная
```

**Ограничить количество коммитов:**

```bash
git log -5           # последние 5
git log -n 10        # последние 10
```

## Фильтрация коммитов

**По автору:**

```bash
git log --author="Иван"
git log --author="ivan@example.com"
git log --author="Иван\|Пётр"  # несколько авторов (regex)
```

**По дате:**

```bash
git log --since="2026-01-01"
git log --until="2026-03-01"
git log --since="2 weeks ago"
git log --since="yesterday"
git log --after="2026-01-01" --before="2026-02-01"
```

**По тексту сообщения:**

```bash
git log --grep="fix"               # коммиты со словом fix
git log --grep="OAuth" -i          # без учёта регистра
git log --grep="Fixes #123"        # коммиты, закрывающие задачу
```

**По изменённому тексту (pickaxe):**

```bash
git log -S "function authenticate"   # коммиты, где строка добавлена/удалена
git log -G "auth.*token"            # коммиты, где строка изменилась (regex)
```

**Коммиты не в слияниях:**

```bash
git log --no-merges
```

## Просмотр изменений в коммитах

**Полный diff каждого коммита:**

```bash
git log -p
git log -p --oneline  # с компактным заголовком
git log -p -- src/auth.js  # diff только для конкретного файла
```

**Статистика изменённых файлов:**

```bash
git log --stat
# feat: добавлена OAuth2 авторизация
#  src/auth.js   | 45 ++++++++++++++
#  tests/auth.js | 30 ++++++++++
#  2 files changed, 75 insertions(+)
```

**Только имена изменённых файлов:**

```bash
git log --name-only
git log --name-status  # с пометками A(added)/M(modified)/D(deleted)
```

## Просмотр коммитов конкретного файла

```bash
# История изменений файла
git log src/auth.js

# С отслеживанием переименований
git log --follow src/auth.js

# Diff каждого коммита для файла
git log -p src/auth.js

# Компактный список коммитов файла
git log --oneline src/auth.js
```

## Диапазоны коммитов

```bash
# Коммиты в feature, но не в main
git log main..feature

# Коммиты в обеих ветках, но не в общей истории
git log main...feature

# Коммиты начиная с определённого хеша
git log a1b2c3d..HEAD

# Последний коммит был добавлен N коммитов назад
git log HEAD~5..HEAD
```

## Продвинутые возможности

**Поиск коммита, внёсшего строку (git blame + log):**

```bash
# Сначала найдём строку
git blame src/auth.js

# Потом изучим коммит
git show a1b2c3d
```

**Реверсный порядок:**

```bash
git log --reverse  # от старых к новым
```

**Показать только один коммит:**

```bash
git log -1          # последний коммит
git log -1 HEAD~5   # пятый с конца
```

**Статистика по авторам:**

```bash
git shortlog       # коммиты сгруппированы по автору
git shortlog -s    # только количество коммитов
git shortlog -sn   # с сортировкой
```

## Практические примеры

```bash
# Просмотр последних 5 коммитов
git log --oneline -5

# Красивый граф всех веток
git log --graph --oneline --all --decorate

# Коммиты автора Иван за последний месяц
git log --author="Иван" --since="1 month ago" --oneline

# Поиск коммитов с исправлениями
git log --grep="fix" --oneline

# Что изменилось в конкретном файле
git log -p -- src/main.js

# Коммиты между двумя ветками
git log main..feature --oneline

# Красивый формат с датой
git log --pretty=format:"%h %ad | %s%d [%an]" --date=short

# Только последние слияния
git log --merges --oneline -10
```

## Полезные алиасы

```bash
# Добавить удобные алиасы в .gitconfig
git config --global alias.lg "log --graph --oneline --all --decorate"
git config --global alias.ll "log --oneline -20"
git config --global alias.lp "log -p --oneline"

# Использование:
git lg   # красивый граф
git ll   # последние 20 в компактном виде
git lp   # с диффами
```

## Часто задаваемые вопросы

**Как выйти из просмотра git log?** Нажмите `q`. Это стандартный `less` pager. `h` — справка, `G` — перейти в конец, `g` — в начало.

**Как увидеть все ветки в git log?** `git log --all --oneline --graph` — покажет все локальные и удалённые ветки.

**Как найти коммит, в котором была добавлена строка?** `git log -S "текст строки"` — покажет коммиты, где эта строка была добавлена или удалена.

**Как вывести коммиты между двумя датами?** `git log --after="2026-01-01" --before="2026-02-01" --oneline`.

**Что означает HEAD в git log?** HEAD — это указатель на текущий коммит. `HEAD~1` — предыдущий, `HEAD~5` — пятый с конца. В выводе log `HEAD -> main` означает, что вы на ветке main, которая указывает на этот коммит.

## Заключение

`git log` — незаменимый инструмент для понимания истории проекта. Начните с `git log --oneline --graph --all` для общего обзора, и `git log -p <file>` для изучения конкретных изменений. Настройте алиасы для часто используемых вариантов.

Для просмотра деталей конкретного коммита используйте [git show]({{< relref "git-show" >}}). Для анализа изменений в рабочей директории и индексе — [git diff]({{< relref "git-diff" >}}).

## По теме
- [Сравнение коммитов в GitLab]({{< relref "gitlab-compare-commits" >}})
- [git blame: кто написал строку]({{< relref "git-blame" >}})

- [git bisect: бинарный поиск бага]({{< relref "git-bisect" >}})
