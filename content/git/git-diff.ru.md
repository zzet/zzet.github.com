---
title: "git diff: полное руководство по сравнению изменений в коде"
description: "Научитесь использовать git diff для просмотра изменений между коммитами, ветками и файлами. Примеры команд для code review и контроля качества."
date: 2026-01-10
lastmod: 2026-01-10
draft: false
slug: "git-diff"
keywords: ["git diff", "сравнение файлов git", "git diff между ветками", "git diff что делает", "git diff --staged", "git diff --cached", "посмотреть изменения git", "git diff между коммитами"]
tags: ["git", "intermediate"]
categories: ["git"]
---

`git diff` — универсальный инструмент для просмотра различий в коде. Он показывает что именно изменилось: в рабочей директории, в индексе, между коммитами, между ветками. Без понимания `git diff` сложно делать качественный code review или отлаживать регрессии.

## Как читать вывод git diff

```
diff --git a/src/auth.js b/src/auth.js
index 123abc..456def 100644
--- a/src/auth.js
+++ b/src/auth.js
@@ -10,7 +10,10 @@
 function login(user) {
-  return authenticate(user.password);
+  const hash = hashPassword(user.password);
+  return authenticate(hash);
+}
+
+function hashPassword(pwd) {
```

Расшифровка: `--- a/src/auth.js` — старая версия файла, `+++ b/src/auth.js` — новая версия. `@@ -10,7 +10,10 @@` — начало изменения (строка 10 в старой, 7 строк; строка 10 в новой, 10 строк). Строки с `-` — удалены, с `+` — добавлены, без знака — контекст.

## Четыре основных сценария

**1. Изменения не в индексе (рабочая директория vs индекс):**

```bash
git diff
```

Показывает что изменено в файлах, но ещё не добавлено через `git add`.

**2. Изменения в индексе (индекс vs последний коммит):**

```bash
git diff --staged
# или синоним:
git diff --cached
```

Показывает что будет в следующем коммите.

**3. Изменения между коммитами:**

```bash
git diff HEAD~1 HEAD   # предыдущий vs текущий
git diff a1b2c3 b4c5d6  # два конкретных коммита
```

**4. Изменения между ветками:**

```bash
git diff main feature   # всё что отличает feature от main
```

## git diff без параметров

```bash
git diff
```

Сравнивает рабочую директорию с индексом. Показывает только то, что НЕ добавлено через `git add`. Если вы всё уже добавили — вывод будет пустым.

## git diff --staged

```bash
git diff --staged
```

Сравнивает индекс с последним коммитом. Показывает именно то, что войдёт в следующий `git commit`. Используйте перед коммитом для проверки.

## Сравнение коммитов

```bash
# Последние два коммита
git diff HEAD~1 HEAD

# Три коммита назад
git diff HEAD~3 HEAD

# Конкретные коммиты
git diff a1b2c3d b4c5d6e

# Коммит vs текущее состояние
git diff a1b2c3d
```

## Сравнение веток

```bash
# Все изменения между ветками
git diff main feature

# Или с явным оператором диапазона
git diff main..feature    # то же самое

# Только изменения из feature (относительно точки ветвления)
git diff main...feature
```

Разница между `..` и `...`: `main..feature` — все различия между ветками сейчас. `main...feature` — только что добавила feature с момента ветвления.

## Diff конкретного файла

```bash
# Изменения в конкретном файле (рабочая директория)
git diff src/auth.js

# Изменения конкретного файла в индексе
git diff --staged src/auth.js

# Конкретный файл между двумя коммитами
git diff HEAD~3 HEAD -- src/auth.js

# Конкретный файл между ветками
git diff main feature -- src/auth.js
```

## Опции форматирования

**Только статистика:**

```bash
git diff --stat
# src/auth.js | 45 ++++++++++++++++++++++--------
# 1 file changed, 30 insertions(+), 15 deletions(-)
```

**Только имена файлов:**

```bash
git diff --name-only
# src/auth.js
# tests/auth.test.js
```

**Имена файлов со статусом:**

```bash
git diff --name-status
# M    src/auth.js      ← Modified
# A    tests/auth.test.js  ← Added
# D    src/old-auth.js  ← Deleted
```

**Diff по словам (удобнее для текста):**

```bash
git diff --word-diff
# [-return authenticate(user.password);-]{+const hash = hashPassword(user.password);+}
```

**Игнорировать пробелы:**

```bash
git diff -w           # игнорировать все пробелы
git diff --ignore-space-change   # игнорировать изменения количества пробелов
```

**Больше контекста:**

```bash
git diff -U10         # показать 10 строк контекста (вместо 3)
```

## Практические примеры

```bash
# Что я изменил, но ещё не добавил
git diff

# Что войдёт в следующий коммит
git diff --staged

# Сравнение последних двух коммитов
git diff HEAD~1 HEAD

# Сравнение между ветками
git diff main..feature

# Статистика изменений
git diff --stat

# Только имена изменённых файлов
git diff --name-only

# Diff конкретного файла
git diff src/main.js

# Разница между файлом в двух коммитах
git diff a1b2c3d b4c5d6e -- src/main.js

# Diff по словам
git diff --word-diff

# Игнорировать пробелы
git diff -w

# История изменений файла за последние 3 коммита
git diff HEAD~3 HEAD -- src/config.js
```

## Часто задаваемые вопросы

**Какая разница между `git diff main..feature` и `git diff main...feature`?** `main..feature` показывает все различия между ветками сейчас. `main...feature` показывает только изменения, которые внесла ветка feature относительно общего предка (точки ветвления).

**Как увидеть только названия изменённых файлов?** `git diff --name-only` — только имена. `git diff --name-status` — имена со статусом (M/A/D).

**Как сравнить файл в двух коммитах?** `git diff <hash1> <hash2> -- path/to/file`.

**Почему git diff не показывает изменения в индексе?** `git diff` без флагов показывает только то, что НЕ в индексе. Для просмотра того, что в индексе: `git diff --staged`.

**Как использовать git diff с визуальным инструментом?** `git difftool` открывает diff в настроенном визуальном инструменте (vimdiff, meld, VS Code). Настройка: `git config --global diff.tool vscode`.

## Заключение

`git diff` — незаменимый инструмент для контроля качества изменений. Привыкните проверять `git diff --staged` перед каждым коммитом — это предотвратит случайные коммиты ненужного кода.

Для просмотра истории коммитов — [git log]({{< relref "git-log" >}}). Для просмотра деталей конкретного коммита — [git show]({{< relref "git-show" >}}). Для работы с индексом — [git add]({{< relref "git-add" >}}).

## По теме
- [Git diff в VS Code (аналог IntelliJ)]({{< relref "vscode-diff-idea" >}})

- [Сравнение коммитов в GitLab]({{< relref "gitlab-compare-commits" >}})
