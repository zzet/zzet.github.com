---
title: "git show: как просмотреть детали коммита и других объектов Git"
description: "Полное руководство по git show. Как просмотреть детали коммита, содержимое файлов из истории, информацию о тегах с примерами команд."
date: 2026-01-29
lastmod: 2026-01-29
draft: false
slug: "git-show"
keywords: ["git show что делает", "git show команда", "просмотр коммита", "git show коммит", "посмотреть изменения коммита git", "git show хэш"]
tags: ["git", "intermediate"]
categories: ["git"]
---

`git show` — команда для просмотра деталей конкретного объекта Git: коммита, тега, файла из истории. Если `git log` даёт список всех коммитов, то `git show` показывает подробности одного конкретного.

## Что такое git show и для чего она нужна

`git show` отображает информацию об объекте Git. Чаще всего используется для просмотра коммитов, но также работает с тегами, деревьями и блобами (содержимым файлов).

Практические применения: изучить, что именно изменилось в коммите; восстановить содержимое файла из прошлого; проверить, что входит в тег; отладить проблему, нашедшуюся через `git bisect`.

## Просмотр коммита

**Последний коммит:**

```bash
git show
# или явно
git show HEAD
```

Вывод содержит: метаданные (автор, дата, сообщение) и diff изменений.

**Конкретный коммит по хешу:**

```bash
git show a1b2c3d
git show a1b2c3d4e5f6789012345678  # полный хеш тоже работает
```

**Предыдущие коммиты:**

```bash
git show HEAD~1    # предыдущий коммит
git show HEAD~5    # пятый коммит с конца
git show HEAD^     # то же что HEAD~1
git show HEAD^^    # то же что HEAD~2
```

**Коммит из другой ветки:**

```bash
git show feature/auth:HEAD
git show main~3
```

Пример вывода `git show`:

```
commit a1b2c3d4e5f6789012345678901234567890abcd
Author: Иван Петров <ivan@example.com>
Date:   Thu Mar 13 15:30:00 2026 +0300

    feat: добавлена OAuth2 авторизация

    Интеграция с Google OAuth2.

diff --git a/src/auth.js b/src/auth.js
index 123abc..456def 100644
--- a/src/auth.js
+++ b/src/auth.js
@@ -10,6 +10,20 @@
+function googleAuth(code) {
+  // ...
+}
```

## Просмотр файла из конкретного коммита

```bash
# Содержимое файла на момент коммита
git show a1b2c3d:src/auth.js

# Содержимое файла в предыдущем коммите
git show HEAD~1:src/auth.js

# Файл из другой ветки
git show main:config.js

# Перенаправить в другой файл (восстановление)
git show HEAD~1:src/auth.js > auth_old.js
```

Это позволяет «заглянуть» в любую версию любого файла без checkout.

## Форматирование вывода

**Только статистика (без diff):**

```bash
git show --stat
# feat: добавлена OAuth2 авторизация
#  src/auth.js   | 45 ++++++++++++++++
#  tests/auth.js | 30 ++++++++++++
#  2 files changed, 75 insertions(+)
```

**Только имена изменённых файлов:**

```bash
git show --name-only
# feat: добавлена OAuth2 авторизация
#
# src/auth.js
# tests/auth.js
```

**Имена файлов со статусом:**

```bash
git show --name-status
# feat: добавлена OAuth2 авторизация
#
# M       src/auth.js        ← Modified
# A       tests/auth.js      ← Added
# D       src/old-auth.js    ← Deleted
```

**Разные форматы вывода:**

```bash
git show --pretty=oneline   # компактный
git show --pretty=fuller    # более детальный (CommitDate, AuthorDate)
git show --pretty=raw       # raw формат для скриптов
```

**Без метаданных (только diff):**

```bash
git show --no-patch  # только метаданные, без diff
```

## git show для тегов

```bash
# Просмотр аннотированного тега
git show v1.0.0

# Вывод:
# tag v1.0.0
# Tagger: Иван Петров <ivan@example.com>
# Date:   Mon Jan 1 00:00:00 2026 +0300
#
# Релиз 1.0.0 - первая стабильная версия
#
# commit a1b2c3d... (tag: v1.0.0)
# ...
```

Для легковесного тега (без аннотации) выводится просто коммит, на который тег указывает.

## Разница между git show и git log -p

`git log -p` показывает diff для каждого коммита в списке — хорошо для просмотра истории. `git show` показывает один конкретный объект — хорошо для изучения деталей.

```bash
# git log -p: последние 3 коммита с диффами
git log -p -3

# git show: конкретный коммит
git show a1b2c3d
```

## Практические примеры

```bash
# Просмотр последнего коммита
git show

# Просмотр конкретного коммита
git show a1b2c3d

# Только статистика без diff
git show a1b2c3d --stat

# Просмотр файла из конкретного коммита
git show a1b2c3d:src/main.js

# Предыдущий коммит
git show HEAD~1

# Коммит из другой ветки
git show develop:config.js

# Просмотр тега
git show v1.0.0

# Только имена изменённых файлов
git show --name-only

# Форматированный вывод
git show --pretty=fuller
```

## Дополнительные варианты использования git show

### Просмотр файла из определённого коммита и сохранение

```bash
# Просмотр содержимого файла как было 5 коммитов назад
git show HEAD~5:src/components/Button.jsx

# Восстановление старой версии файла
git show HEAD~3:src/config.js > src/config.js.old

# Сравнение старой и новой версии файла
git show HEAD~1:README.md > /tmp/old-readme.md
diff /tmp/old-readme.md README.md

# Получение содержимого файла из конкретного тега
git show v1.0.0:src/index.js

# Просмотр файла из другой ветки
git show feature/new-ui:src/App.jsx
```

### Анализ изменений по-файлам

```bash
# Показать только список изменённых файлов
git show --name-only

# Показать статус каждого файла (добавлен/изменён/удалён)
git show --name-status

# Показать статистику с числом добавленных/удалённых строк
git show --stat

# Пример вывода --stat:
# feat: добавлена OAuth2 авторизация
#
#  src/auth.js      | 45 +++++++++++++++++++
#  src/login.js     | 12 +-----
#  tests/auth.test  | 30 +++++++++++++++
#  3 files changed, 87 insertions(+), 5 deletions(-)
```

### Форматирование вывода для специальных случаев

```bash
# Краткий вывод (одна строка на коммит)
git show --oneline

# Полный вывод с метаданными
git show --pretty=fuller
# commit a1b2c3d...
# Author:     John Doe <john@example.com>
# AuthorDate: Mon Mar 10 15:30:00 2026 +0300
# Commit:     Jane Smith <jane@example.com>
# CommitDate: Mon Mar 10 16:00:00 2026 +0300

# Формат для парсинга скриптами
git show --pretty=raw

# Без метаданных, только diff
git show --no-patch
# commit a1b2c3d
# Author: John Doe
# Date: Mon Mar 10 15:30:00 2026
# feat: добавлена OAuth2 авторизация

# С отступом (для обработки в скриптах)
git show --pretty=medium --indent=4
```

## Сравнение git show, git diff и git log

Все три команды показывают изменения, но по-разному:

```bash
# git show: один конкретный коммит
git show a1b2c3d
# Выводит: метаданные коммита + все изменения в этом коммите

# git log -p: последовательность коммитов
git log -p -3
# Выводит: 3 последних коммита с их диффами

# git diff: сравнение двух состояний
git diff HEAD~3 HEAD
# Выводит: все изменения за последние 3 коммита (без метаданных коммитов)

# git diff в текущем рабочем каталоге
git diff src/auth.js
# Выводит: отличия текущего файла от версии в HEAD
```

**Когда использовать какую:**

```bash
git show           # Хочу изучить конкретный коммит
git show <hash>    # Нашёл интересный коммит в истории
git log -p         # Хочу посмотреть развитие файла во времени
git log -p FILE    # Хочу историю одного файла
git diff           # Какие изменения я сделал?
git diff A B       # Разница между двумя коммитами/ветками
```

## Использование git show с фильтрацией

```bash
# Показать только изменения в определённом файле
git show a1b2c3d -- src/auth.js

# Показать только добавленные строки (без удалённых)
git show a1b2c3d | grep "^+" | grep -v "^+++"

# Вывести в цвете (может быть не активно по умолчанию)
git show --color a1b2c3d

# Отключить раскраску
git show --no-color a1b2c3d

# Использовать патч формат для применения в другой ветке
git show a1b2c3d > feature.patch
git apply feature.patch

# Или прямое применение
git show a1b2c3d | git apply
```

## Работа с несколькими коммитами

```bash
# Показать несколько коммитов подряд
git show a1b2c3d e5f6a7b

# Диапазон коммитов (все между двумя)
git show a1b2c3d..e5f6a7b

# Все коммиты от одного до другого включительно
git show a1b2c3d^..e5f6a7b

# Последние 5 коммитов с диффами
for commit in $(git log --oneline -5 | awk '{print $1}' | tac); do
    git show "$commit"
done
```

## Практические примеры для отладки

```bash
# Нашли баг, нужно понять какой коммит его внёс
git log --oneline -S "старый код"
# a1b2c3d feat: refactored authentication

# Посмотреть именно этот коммит
git show a1b2c3d

# Если нужно понять WHO это написал
git show -p a1b2c3d | grep -A 5 -B 5 "интересная строка"

# Когда эта функция была удалена?
git log --all -S "function_name()" -- src/file.js
# Найдёт коммиты где эта функция появилась/удалилась

# Посмотреть последний коммит который трогал файл
git log -n 1 -- src/important-file.js
# Затем показать этот коммит
git show $(git log -n 1 --pretty=format:"%H" -- src/important-file.js)
```

## Часто задаваемые вопросы

**Какая разница между git show и git log -p?** `git log -p` показывает цепочку коммитов с диффами. `git show` — один конкретный объект (коммит, тег, файл). Для быстрого просмотра одного коммита — `git show`, для изучения истории — `git log -p`.

**Как посмотреть файл из определённого коммита?** `git show <hash>:<путь_к_файлу>`. Например: `git show HEAD~1:src/auth.js`.

**Что означает HEAD в git show?** HEAD — текущий коммит. `HEAD~1` — предыдущий. `HEAD~N` — N коммитов назад. `HEAD^` — то же что `HEAD~1`.

**Как использовать git show для просмотра тегов?** `git show v1.0.0` — выведет информацию о теге (если аннотированный) и связанный коммит.

**Как восстановить файл из прошлого коммита?** `git show HEAD~1:src/file.js > file.js` — перезапишет текущий файл содержимым из предыдущего коммита. Альтернатива: `git checkout HEAD~1 -- src/file.js`.

**Как увидеть какие файлы изменялись в коммите?** `git show --name-only` или `git show --name-status` для получения дополнительных деталей об удалённых/изменённых файлах.

## Заключение

`git show` — простой но мощный инструмент. Самые частые сценарии: `git show HEAD` для проверки последнего коммита, `git show <hash>:<file>` для просмотра старой версии файла, `git show --stat` для быстрого обзора изменений.

Для изучения истории — [git log]({{< relref "git-log" >}}). Для анализа текущих изменений — [git diff]({{< relref "git-diff" >}}).
