---
title: "Как просмотреть коммиты в ветке Git: полное руководство"
description: "Узнайте, как просмотреть коммиты в ветке Git. git log, фильтры, форматирование, красивая визуализация веток с примерами."
date: 2026-02-12
lastmod: 2026-02-12
draft: false
slug: "kommity-v-vetke"
keywords: ["посмотреть коммиты в ветке git", "коммиты в ветке", "git branch commits", "git log ветка", "список коммитов ветки git", "git log branch name"]
tags: ["git", "beginner", "branch"]
categories: ["git"]
---

Просмотр коммитов в ветке — базовая операция при code review, отладке и понимании истории изменений. Git предоставляет несколько способов это сделать, от простого списка до красивого дерева веток.

## Просмотр всех коммитов в текущей ветке

```bash
# Полный формат
git log

# Компактный (самый используемый)
git log --oneline

# Последние 10 коммитов
git log -10 --oneline

# С подробной статистикой изменений
git log --stat
```

Пример вывода `git log --oneline`:

```
a1b2c3d (HEAD -> feature/auth) feat: добавлена OAuth2 авторизация
b2c3d4e fix: исправлена ошибка в форме входа
c3d4e5f refactor: упрощена логика проверки токенов
d4e5f6g test: добавлены тесты для авторизации
e5f6g7h docs: обновлена документация по API
```

## Просмотр коммитов, уникальных для ветки

Самое важное при работе с ветками — видеть только те коммиты, которых нет в основной ветке:

```bash
# Коммиты в текущей ветке, которых нет в main
git log main..HEAD --oneline

# Коммиты в feature, которых нет в main
git log main..feature --oneline

# Короткая запись: коммиты не в main
git log main.. --oneline
```

Это показывает именно то, что будет добавлено при слиянии feature в main.

## Просмотр коммитов другой ветки

```bash
# История конкретной ветки
git log feature/auth --oneline

# История удалённой ветки
git log origin/feature/auth --oneline

# История ветки с диффом
git log -p feature/auth

# Количество коммитов в ветке
git log feature/auth --oneline | wc -l
```

## Визуальный граф веток

```bash
# Граф всех веток
git log --graph --oneline --all

# Граф с именами веток
git log --graph --oneline --all --decorate
```

Пример вывода:

```
* a1b2c3d (HEAD -> feature) feat: авторизация
* b2c3d4e fix: форма входа
| * c3d4e5f (main) chore: обновлены зависимости
|/
* d4e5f6g Initial commit
```

## Фильтрация коммитов в ветке

```bash
# Коммиты автора
git log feature/auth --author="Иван" --oneline

# Коммиты за последнюю неделю
git log feature/auth --since="1 week ago" --oneline

# Коммиты с определённым словом в сообщении
git log feature/auth --grep="fix" --oneline

# Коммиты, изменившие конкретный файл
git log feature/auth --oneline -- src/auth.js
```

## Сравнение коммитов между ветками

```bash
# Коммиты в main, но не в feature (что пропустили)
git log feature..main --oneline

# Коммиты в feature, но не в main (что добавили)
git log main..feature --oneline

# Коммиты в обоих направлениях
git log main...feature --oneline
```

## git cherry: коммиты не слитые в upstream

`git cherry` показывает какие коммиты в текущей ветке не были слиты в родительскую ветку:

```bash
# Посмотреть коммиты feature которых нет в main
git cherry main
# + abc1234 feat: add pagination
# + def5678 fix: button alignment
# - ghi9012 refactor: optimize

# Знак + означает что коммит есть в feature но не в main
# Знак - означает что есть эквивалент в main (но с другим хешем — cherry-pick)

# С выводом сообщений коммитов
git cherry main -v
```

## Просмотр коммитов, которые будут слиты (merges)

```bash
# Показать только merge-коммиты
git log --merges --oneline

# Показать коммиты которые НЕ являются merge
git log --no-merges --oneline

# История feature с только реальными коммитами (без merges)
git log feature --no-merges --oneline
```

## Когда был слит коммит в main

```bash
# Найти когда коммит был merge'ен в main
git branch --contains abc1234
# * feature
#   main        ← коммит в main
#   develop

# Или с подробностями
git log --oneline --all | grep abc1234
```

## Просмотр изменений в коммитах (git log -p)

```bash
# Показать все изменения в коммитах
git log -p

# Последние 3 коммита с diff
git log -p -3

# Коммиты конкретного файла с diff
git log -p -- src/auth.js

# Только статистика изменений без полного diff
git log --stat

# Компактный формат: строки добавлены/удалены
git log --shortstat --oneline
```

Пример вывода `git log -p`:

```
commit abc1234
Author: John Doe <john@example.com>
Date:   Mon Mar 15 10:30:00 2026 +0300

    feat: add user registration

diff --git a/src/auth.js b/src/auth.js
index old..new 100644
--- a/src/auth.js
+++ b/src/auth.js
@@ -1,3 +1,10 @@
+export function register(email, password) {
+  // registration logic
+}
```

## Просмотр коммитов по автору (shortlog)

```bash
# Коммиты сгруппированные по автору
git shortlog

# С количеством коммитов
git shortlog -sn

# Пример вывода:
# John Doe (15):
#   feat: add dashboard
#   fix: button styling
# ...
#
# Jane Smith (8):
#   refactor: optimize queries
#   docs: update README

# Коммиты конкретного автора
git log --author="John" --oneline

# За период времени
git log --author="Jane" --since="2 weeks ago" --oneline
```

## Выбор красивого формата для git log

```bash
# Компактный цветной формат
git log --pretty=format:"%h - %an <%ae>, %ar : %s" --graph --all

# С датой и веткой
git log --pretty=format:"%h %ad | %s" --date=short

# С информацией о файлах
git log --pretty=format:"%h %s" --name-only --oneline

# Сохранить в alias для удобства (в ~/.gitconfig)
[alias]
    lg = log --graph --oneline --all --decorate
    lgs = log --shortlog --oneline
```

## Практические примеры

```bash
# Просмотр всех коммитов в текущей ветке
git log --oneline

# Последние 5 коммитов
git log --oneline -5

# Коммиты, уникальные для ветки (не в main)
git log main..HEAD --oneline

# Коммиты конкретной ветки
git log feature-branch --oneline

# Коммиты удалённой ветки
git log origin/feature --oneline

# Граф всех веток
git log --graph --oneline --all

# Коммиты с diff
git log -p --oneline -3

# Красивый формат
git log --pretty=format:"%h - %an, %ar : %s"

# История конкретного файла в ветке
git log --oneline -- path/to/file.js

# Коммиты за последнюю неделю
git log --since="1 week ago" --oneline

# Коммиты одного автора
git shortlog -sn

# Только merge-коммиты
git log --merges --oneline

# Коммиты которые не в main
git log main.. --oneline
```

## Часто задаваемые вопросы

**Как вывести только коммиты текущей ветки?** `git log --oneline` показывает все коммиты текущей ветки. Для просмотра только уникальных (которых нет в main): `git log main..HEAD --oneline`. Это очень полезно при подготовке pull request.

**Как посмотреть коммиты которые не в master?** `git log master..HEAD --oneline` покажет коммиты текущей ветки которых нет в master. Или `git log master..branch-name --oneline` для любой ветки. Это показывает ровно то что будет добавлено при merge.

**Как посчитать количество коммитов в ветке?** `git log --oneline | wc -l` показывает все коммиты. `git log main..HEAD --oneline | wc -l` — только уникальные для текущей ветки относительно main.

**Как просмотреть коммиты удалённой ветки?** Сначала `git fetch origin` для обновления локальной копии информации о удалённых ветках. Затем `git log origin/feature --oneline`. Или одной командой: `git log -u origin/feature --oneline`.

**Как вывести коммиты в красивом формате?** `git log --graph --oneline --all --decorate` показывает граф всех веток с красивой структурой. `git log --pretty=format:"%h %ad | %s" --date=short` выводит в табличном формате. Сохраните как alias для удобства.

**Как посмотреть только собственные коммиты?** `git log --author="Ваше Имя" --oneline` или `git log --author="$USER" --oneline`. Для email-а: `git log --author="your@email.com" --oneline`.

**Чем отличается git log main..feature от git log main...feature?** `main..feature` показывает коммиты в feature которых нет в main. `main...feature` показывает коммиты в обоих направлениях (в feature но не в main И в main но не в feature). Первый вариант используется чаще.

## Статистика коммитов за период времени

```bash
# Коммиты за последний месяц
git log --since="1 month ago" --oneline

# Коммиты за конкретный период
git log --since="2026-01-01" --until="2026-03-15" --oneline

# Количество коммитов в месяц
git log --since="2026-01-01" --until="2026-03-15" --oneline | wc -l

# Статистика: сколько коммитов в неделю
git log --since="1 month ago" --pretty=format:"%ad" --date=short | sort | uniq -c
```

## Экспорт логов в разные форматы

```bash
# Экспорт в CSV
git log --pretty=format:"%h,%an,%ae,%ad,%s" --date=short > commits.csv

# Экспорт в JSON (можно обработать через jq)
git log --pretty=format:"{%n  \"hash\": \"%H\",%n  \"author\": \"%an\",%n  \"message\": \"%s\"%n}," | head -c -1 | sed -e 's/,$//'

# Сохранить в файл текстовый формат
git log --oneline > commits.txt

# С подробностями
git log --stat > commits_detailed.txt
```

## Просмотр истории конкретного участка кода

```bash
# Показать коммиты которые редактировали конкретные строки
git log -L10,20:src/auth.js
# Показывает историю строк 10-20 в файле src/auth.js

# С diff для каждого коммита
git log -L10,20:src/auth.js -p
```

## Анализ того кто написал код (blame)

```bash
# Показать для каждой строки кто её написал и в каком коммите
git blame src/auth.js

# Вывод: хеш коммита, автор, дата, номер строки, код

# Только определённые строки
git blame -L10,20 src/auth.js

# Игнорировать whitespace изменения
git blame -w src/auth.js

# Показать в долгом формате
git blame -l src/auth.js
```

## Визуализация истории в графическом интерфейсе

Если текстовый вывод недостаточен, используйте GUI:

```bash
# GitKraken (графический клиент)
# Файл → Open → выбрать репо
# Видит всю историю в виде красивого графа

# gitk (встроенный в Git)
gitk --all &

# tig (текстовый UI для истории)
tig --all

# Используя Git в IDE (VS Code, IntelliJ)
# Source Control вкладка показывает историю визуально
```

## Интеграция с системами отслеживания ошибок

Часто репо привязано к issue tracker'ам:

```bash
# Коммиты связанные с Jira issue
git log --grep="PROJ-" --oneline

# Коммиты для GitHub issue
git log --grep="#123" --oneline

# Коммиты с тегами
git log --grep="\[bug\]" --oneline

# Фильтр по нескольким словам
git log --grep="feature\|enhancement" --oneline
```

## Заключение

Для обычного просмотра — `git log --oneline`. Для code review — `git log main..HEAD --oneline`. Для общего обзора всего проекта — `git log --graph --oneline --all`. Для анализа за период времени — `git log --since="1 month ago"`. Для понимания кто изменил строку — `git blame filename`.

Подробнее о команде git log — [полное руководство по git log]({{< relref "git-log" >}}). Для работы с ветками — [ветки в Git]({{< relref "vetki-v-git" >}}).
