---
title: "Как посмотреть изменения в Git: полное руководство"
description: "Полное руководство по просмотру изменений в Git. Команды git status, git diff, git log с примерами для начинающих."
date: 2026-02-09
lastmod: 2026-02-09
draft: false
slug: "kak-posmotret-izmeneniya-git"
keywords: ["посмотреть изменения git", "как посмотреть что изменилось", "git статус", "git просмотр изменений", "git diff changes", "посмотреть незакоммиченные изменения", "как посмотреть изменения в файле git", "как посмотреть изменения в репозитории git", "git log изменения файла"]
tags: ["git", "beginner"]
categories: ["git"]
---

В Git есть несколько команд для просмотра изменений, и каждая отвечает на свой вопрос. `git status` — что сейчас изменено? `git diff` — что именно изменилось в коде? `git log` — что менялось в прошлом? Эта статья объясняет, когда использовать каждую команду.

## git status: общий обзор состояния

`git status` — первая команда, которую нужно запускать для понимания ситуации:

```bash
git status
```

Пример вывода:

```
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   src/auth.js        ← в индексе (будет в коммите)

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
        modified:   src/utils.js       ← изменён, не в индексе

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        src/new-feature.js             ← новый файл, не отслеживается
```

Три категории: «Changes to be committed» — в индексе, войдёт в следующий коммит; «Changes not staged» — изменено, но не добавлено; «Untracked files» — новые файлы, Git не знает о них.

**Краткий формат:**

```bash
git status -s
# M  src/auth.js    ← M в первой колонке = в индексе
#  M src/utils.js   ← M во второй колонке = не в индексе
# ?? src/new.js     ← ?? = не отслеживается
```

## git diff: что именно изменилось

`git diff` показывает точные изменения в коде (построчно):

```bash
# Изменения не в индексе
git diff

# Изменения в индексе (что войдёт в коммит)
git diff --staged

# Только статистику (без построчного diff)
git diff --stat
```

Типичный вывод:

```diff
--- a/src/utils.js
+++ b/src/utils.js
@@ -15,6 +15,8 @@
 function formatDate(date) {
-  return date.toISOString();
+  const options = { year: 'numeric', month: 'long', day: 'numeric' };
+  return date.toLocaleDateString('ru-RU', options);
 }
```

Строки с `-` удалены, с `+` добавлены.

**Быстрая проверка перед коммитом:**

```bash
git status          # что изменено?
git diff --staged   # что именно войдёт в коммит?
git commit -m "..."
```

## git log: история прошлых изменений

`git log` показывает историю коммитов:

```bash
# Полная история
git log

# Компактный формат
git log --oneline -10

# С графом веток
git log --graph --oneline --all
```

Для просмотра изменений в коммитах:

```bash
# Что изменилось в последнем коммите
git show

# История конкретного файла
git log --oneline src/auth.js

# Изменения в файле по коммитам
git log -p src/auth.js
```

## Просмотр изменений в конкретном файле

```bash
# Что изменилось в файле (не в индексе)
git diff src/main.js

# История изменений файла
git log --oneline src/main.js

# Полный diff истории файла
git log -p src/main.js

# Кто и когда изменил каждую строку
git blame src/main.js
```

## Просмотр изменений, не отправленных на сервер

```bash
# Что запушим (коммиты, которых нет на сервере)
git log origin/main..HEAD

# Что запушим в виде diff
git diff origin/main HEAD
```

## git diff с дополнительными опциями

Более продвинутые варианты git diff:

```bash
# Сравнить две ветки
git diff main feature/new-feature
git diff main..feature/new-feature  # такой же результат
git diff main...feature/new-feature # только изменения в feature ветке

# Сравнить два коммита
git diff abc1234 def5678

# Сравнить коммит с parent
git diff HEAD~1 HEAD
git diff abc1234^..abc1234  # коммит abc1234 против его parent

# Diff только для конкретного файла
git diff src/auth.js
git diff main..feature src/auth.js  # между ветками, но только файл

# Игнорировать пробелы
git diff -w      # игнорировать все пробелы
git diff -b      # игнорировать changes в пробелах

# Word-level diff (вместо line-level)
git diff --word-diff
git diff --word-diff-regex='[^[:space:]]+'

# Только имена файлов (без содержимого)
git diff --name-only
git diff --name-status  # с статусом (M=modified, A=added, D=deleted)

# Статистика изменений
git diff --stat
git diff --stat main..feature

# Проценты изменений
git diff --diffstat

# Числовые изменения
git diff -U0  # 0 lines of context (только что изменилось, без контекста)
git diff -U10 # 10 lines of context
```

## git show: просмотр конкретного коммита

`git show` — специальная команда для просмотра одного коммита:

```bash
# Показать последний коммит
git show

# Показать конкретный коммит
git show abc1234

# Показать определённый файл из конкретного коммита
git show abc1234:src/main.js

# Без diff, только информация о коммите
git show --stat abc1234

# Показать однострочное имя коммита
git show --oneline abc1234

# Format вывода
git show --format=medium abc1234
git show --format=fuller abc1234
git show --format=raw abc1234
```

## git log -p: история с дифами

История всех изменений в файле:

```bash
# История файла со всеми дифами
git log -p src/auth.js

# Последние 3 коммита с дифами
git log -p -3

# История функции (требует -L опцию)
git log -L :functionName:src/file.js
git log -L 10,20:src/file.js  # строки 10-20 файла

# История с графом и дифами
git log -p --graph --oneline --all

# История одного автора
git log -p --author="John Doe"

# История за период времени
git log -p --since="2024-01-01" --until="2024-12-31"
```

## git blame: кто и когда изменил каждую строку

`git blame` показывает для каждой строки: кто её добавил, когда, в каком коммите:

```bash
# Посмотреть кто изменил каждую строку
git blame src/main.js

# Вывод:
# 7c1234ab src/main.js (Alice 2024-01-15 10:30:45 +0000  1) function start() {
# 7c1234ab src/main.js (Alice 2024-01-15 10:30:45 +0000  2)   console.log('starting');
# 8d5678ef src/main.js (Bob   2024-02-20 14:20:10 +0000  3)   init();
# 8d5678ef src/main.js (Bob   2024-02-20 14:20:10 +0000  4) }

# Только хеши коммитов
git blame -l src/main.js

# Считать пробелы и пунктуацию (не просто пробелы)
git blame -w src/main.js

# Показать дополнительную информацию
git blame -e src/main.js  # email вместо имени
git blame --date=short src/main.js

# Для определённых строк
git blame -L 10,20 src/main.js

# Интерактивно с git-log
git blame -C src/main.js  # показать если строка скопирована из другого файла
```

## Сравнение веток и коммитов

Быстро посмотреть что отличается:

```bash
# Какие коммиты в feature но не в main
git log main..feature --oneline

# Какие коммиты в main но не в feature
git log feature..main --oneline

# Кол-во коммитов
git log main..feature --oneline | wc -l

# Все коммиты между ветками в обе стороны
git log main...feature --oneline

# Файлы которые отличаются
git diff main feature --name-only

# С статусом
git diff main feature --name-status
# M  src/auth.js  (modified)
# A  src/new.js   (added)
# D  src/old.js   (deleted)
```

## Визуальные инструменты для diff

Настройка внешних инструментов:

```bash
# VS Code как difftool
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'

# Использование
git difftool src/auth.js

# Meld (Linux)
git config --global diff.tool meld
git difftool src/auth.js

# KDiff3
git config --global diff.tool kdiff3
git difftool src/auth.js

# Beyond Compare
git config --global diff.tool bc
git config --global difftool.bc.cmd 'bcomp $LOCAL $REMOTE'
```

## Интерактивный просмотр в IDE

Все современные IDE поддерживают Git визуально:

**VS Code:**
```
- Ctrl+Shift+G: Source Control panel
- Видны все изменённые файлы
- Клик на файл → diff боком
- Встроенный merge conflict resolver
- Timeline view (история файла)
```

**JetBrains (IntelliJ, PyCharm, WebStorm):**
```
- Alt+9: Git tool window
- Или: View → Tool Windows → Git
- Visual diff с подсветкой синтаксиса
- Blame annotations (кто и когда)
- Merge conflict resolution UI
```

**Sublime Text:**
```
- Package: GitGutter (shows diff in gutter)
- Package: Git (Git commands)
- Или просто запустить: git diff в терминале
```

## Проверка перед commit

Типичный workflow для проверки перед коммитом:

```bash
# 1. Посмотреть общий статус
git status

# 2. Посмотреть точные изменения (не staged)
git diff

# 3. Добавить в индекс
git add src/auth.js

# 4. Посмотреть что именно будет в коммите
git diff --staged

# 5. Если всё ОК
git commit -m "feat: add authentication"

# 6. Проверить что залетело в коммит
git show
```

## Практические примеры

```bash
# Состояние репозитория
git status

# Краткий формат статуса
git status -s

# Просмотр точных изменений в файлах
git diff

# Изменения, готовые к коммиту
git diff --staged

# Просмотр истории коммитов
git log --oneline -10

# История коммитов с изменениями
git log -p -3

# Информация о конкретном коммите
git show a1b2c3d

# Просмотр только добавленных и удалённых строк
git diff --stat

# История конкретного файла
git log -p src/main.js

# Просмотр веток и их позиций
git branch -v

# Изменения, которые не отправлены на сервер
git log origin/main..HEAD

# Статус с подробной информацией
git status --porcelain

# Просмотр изменений в конкретном диапазоне
git log -p main..feature/new

# Всё что изменилось на текущей ветке
git diff main...HEAD

# История удаления файла
git log --diff-filter=D --summary | grep delete

# Показать коммит когда файл был последний раз изменён
git log -1 --format=%H -- src/main.js

# Просмотр веток которые уже слиты
git branch --merged

# Просмотр веток которые ещё не слиты
git branch --no-merged
```

## Быстрые команды для повседневной работы

Шпаргалка для постоянного использования:

```bash
# Перед commit: что добавится?
git diff --cached     # или --staged (в новых версиях)

# После commit: что залетело?
git show              # последний коммит
git log -1 -p         # то же самое

# Что я сделал сегодня?
git log --since="8am" --until="now" --oneline

# Чего не отправить на сервер?
git log origin/main..HEAD

# Кто последний менял файл?
git log -1 --format=%an -- src/main.js

# Когда файл был создан?
git log --follow --format=%H -- src/main.js | tail -1

# Размер изменений
git diff --stat main..HEAD

# Статус как для скрипта
git status --porcelain > status.txt

# Виды изменений
git diff --name-status main..HEAD

# Просмотр неcommented кода (WIP коммитов)
git log --oneline --grep="WIP" --grep="TODO"

# История конфликтов merge
git log --oneline | grep Merge
```

## Анализ и статистика

Получить информацию о разработке:

```bash
# Кол-во коммитов по авторам
git shortlog -sn

# Изменённые строки по авторам
git shortlog -sn -e

# Статистика по файлам (добавлено/удалено)
git log --stat | grep "files changed"

# Самые часто изменяемые файлы
git log --oneline --name-only | sort | uniq -c | sort -rn

# Анализ merge коммитов (разрешение конфликтов)
git log --merges --oneline

# Когда была последняя активность на ветке
git for-each-ref --sort=-committerdate refs/heads/ --format='%(refname:short) %(committerdate)'

# Список всех веток отсортированных по дате
git branch -a --sort=-version:refname --sort=-committerdate
```

## Часто задаваемые вопросы

**Как увидеть, что изменилось в файле перед коммитом?** `git diff <файл>` — для не добавленных изменений. `git diff --staged <файл>` — для уже добавленных в индекс.

**Какая команда показывает все изменения в репозитории?** `git status` — общий обзор. `git diff` — детали не-staged изменений. `git diff --staged` — детали staged. Все вместе дают полную картину.

**Как увидеть историю изменений конкретного файла?** `git log --oneline src/file.js` — список коммитов. `git log -p src/file.js` — с диффами. `git blame src/file.js` — построчно с авторами.

**Что означают разные символы в git status -s?** В первой колонке — статус в индексе: `M` изменён, `A` добавлен, `D` удалён, `R` переименован. Во второй колонке — статус в рабочей директории. `??` — не отслеживается.

**Как просмотреть изменения в удалённой ветке?** `git fetch origin` для обновления информации, затем `git diff main origin/main` для сравнения.

## Заключение

Для понимания состояния репозитория достаточно трёх команд: `git status` (что изменено), `git diff --staged` (что войдёт в коммит), `git log --oneline` (история). Привыкните запускать их регулярно — это сделает работу с Git осознанной.

Детальнее о каждой команде: [git diff]({{< relref "git-diff" >}}), [git log]({{< relref "git-log" >}}), [git add]({{< relref "git-add" >}}).
