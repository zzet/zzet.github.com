---
title: "git switch и git restore: современная замена git checkout (Git 2.23+)"
description: "git switch заменяет git checkout для веток. git restore — для сброса изменений. Полный синтаксис, примеры, сравнение с git checkout."
date: 2026-01-16
lastmod: 2026-03-14
draft: false
slug: "git-switch"
keywords: ["git switch", "git switch ветка", "git switch remote branch", "git restore", "git switch vs checkout", "git switch to commit"]
tags: ["git", "beginner"]
categories: ["git"]
---

Git 2.23, выпущенный в августе 2019 года, представил две новые команды, которые разделили функциональность перегруженной команды `git checkout`:

- **`git switch`** — для переключения между ветками и создания новых веток
- **`git restore`** — для восстановления файлов и сброса изменений

До этого одна команда `git checkout` выполняла обе функции, что было запутанным и противоинтуитивным. Если вы работаете с Git 2.23 или новее, **рекомендуется использовать `git switch` и `git restore`** вместо `git checkout`.

В этой статье мы разберём обе команды подробно, покажем все синтаксические варианты, и поясним когда использовать какую.

## git switch: переключение между ветками

Основное назначение `git switch` — это переключение между ветками в вашем репозитории.

### Переключиться на существующую ветку

```bash
git switch main
git switch develop
git switch feature-auth
```

Это самый простой случай. Если ветка существует локально, просто передайте её имя. Для более подробного понимания работы с ветками см. {{< relref "pereklyuchitsya-mezhdu-vetkami" >}}.

**Полный пример:**
```bash
$ git branch
* main
  feature-login
  feature-auth

$ git switch feature-auth
Switched to branch 'feature-auth'

$ git branch
  main
* feature-auth
  feature-login
```

### Создать новую ветку и переключиться на неё

```bash
# Создать новую ветку и сразу переключиться
git switch -c new-feature

# Или с полным флагом
git switch --create new-feature

# Создать на основе другой ветки
git switch -c new-feature develop
```

**Пример с базовой веткой:**
```bash
# Вы на main, но хотите создать feature на основе develop
$ git switch -c feature-new-design develop
Switched to a new branch 'feature-new-design' tracking branch 'develop'

# Теперь вы на новой ветке, основанной на develop
$ git log --oneline -n 3
abc1234 Latest commit from develop
def5678 Another commit
ghi9012 Older commit
```

### Отслеживать удалённую ветку (tracking branch)

Когда вы переключаетесь на удалённую ветку, Git автоматически создаёт локальную ветку которая её отслеживает:

```bash
# Переключиться на удалённую ветку origin/feature-x
# Git автоматически создаст локальную feature-x
git switch feature-x

# Или явно указать удалённую ветку
git switch --track origin/feature-payment

# Более короткий вариант
git switch -t origin/feature-payment
```

**Практический пример:**
```bash
# Вы получили fetch обновлений от всех удалённых веток
$ git fetch

# Видите что появилась новая ветка в origin
$ git branch -r
  origin/main
  origin/develop
  origin/feature-new-payment

# Переключаетесь на неё
$ git switch feature-new-payment
branch 'feature-new-payment' set up to track 'origin/feature-new-payment'
Switched to a new branch 'feature-new-payment'

# Теперь у вас есть локальная ветка которая отслеживает удалённую
```

### Переключиться на предыдущую ветку

```bash
# Вернуться на ветку на которой вы были перед этой
git switch -
```

Это удобно для быстрого переключения между двумя ветками:

```bash
$ git branch
* feature-auth
  main

$ git switch -
Switched to branch 'main'

$ git switch -
Switched to branch 'feature-auth'

$ git switch -
Switched to branch 'main'
```

### Обработка необработанных изменений

Если у вас есть изменения в файлах, которые конфликтуют с целевой веткой, Git не позволит вам переключиться:

```bash
$ git status
On branch feature-auth
Changes not staged for commit:
  modified:   src/auth.js

$ git switch main
error: Your local changes to 'src/auth.js' would be overwritten by checkout.
Please commit your changes or stash them before you switch branches.
```

У вас есть несколько вариантов:

```bash
# Вариант 1: создать коммит ваших изменений
git add .
git commit -m "Work in progress on auth"
git switch main

# Вариант 2: отложить изменения (сохранить в stash)
git stash
git switch main
git stash pop  # Вернуть изменения когда нужно

# Вариант 3: откатить изменения (опасно!)
git restore src/auth.js
git switch main
```

## git switch к коммиту (detached HEAD)

Хотя основное назначение `git switch` — работа с ветками, её также можно использовать для перехода к конкретному коммиту. Это перейдёт вас в "detached HEAD" состояние:

```bash
# Переключиться на конкретный коммит
git switch --detach abc1234

# Более короткий вариант
git switch -d abc1234

# Или переключиться к определённому релизу
git switch --detach v1.2.3
```

### Что такое detached HEAD?

Обычно вы находитесь "на ветке" (HEAD указывает на имя ветки). В detached HEAD состоянии HEAD указывает прямо на коммит:

```bash
# На нормальной ветке
$ git switch main
Switched to branch 'main'

$ git status
On branch main

# В detached HEAD
$ git switch -d abc1234
Switched to a detached HEAD state at abc1234

$ git status
HEAD detached at abc1234
```

### Когда это полезно

Detached HEAD полезен когда вы хотите:

```bash
# Посмотреть как выглядел проект в старом коммите
git switch -d abc1234
# смотрите файлы, запускаете старый код
# потом возвращаетесь

# Если нашли что-то важное, создаёте ветку
git switch -c investigate-old-bug

# Или возвращаетесь на нормальную ветку
git switch main
```

**Полный пример:**
```bash
$ git log --oneline -n 5
d7f8e9a Current version (HEAD -> main)
c4b3a2d Added feature X
b1c2d3e Refactored database
a1b2c3d Two weeks ago

# Хотим посмотреть как выглядел проект в коммите a1b2c3d
$ git switch -d a1b2c3d
Switched to a detached HEAD state at a1b2c3d

$ git status
HEAD detached at a1b2c3d

# Смотрим, запускаем, тестируем

# Нашли важный файл который был потерян
$ git switch -c recover-old-feature

# Делаем необходимые коммиты
$ git add .
$ git commit -m "Restore important old feature"

# Потом смержим в main
$ git switch main
$ git merge recover-old-feature

# Если ничего не нужно было, просто возвращаемся
$ git switch main
$ git branch -D some-detached-work
```

## git restore: восстановление файлов

`git restore` заменяет функциональность `git checkout` для работы с файлами. Это команда для отката изменений в файлах.

### Отменить изменения в файле

```bash
# Отменить все изменения в одном файле
git restore src/auth.js

# Отменить изменения в нескольких файлах
git restore src/auth.js src/user.js

# Отменить все изменения в текущей директории
git restore .
```

**Пример:**
```bash
$ git status
On branch main
Changes not staged for commit:
  modified:   src/auth.js
  modified:   src/user.js

$ git restore src/auth.js

$ git status
On branch main
Changes not staged for commit:
  modified:   src/user.js
```

### Отменить staged изменения

Если вы выполнили `git add` но хотите отменить это:

```bash
# Отменить staging для одного файла
git restore --staged src/auth.js

# Или для всех файлов
git restore --staged .
```

**Полный пример:**
```bash
$ git status
On branch main
Changes to be committed:
  modified:   src/auth.js
  new file:   src/validation.js

# Отменяем staging для файла
$ git restore --staged src/auth.js

$ git status
On branch main
Changes to be committed:
  new file:   src/validation.js

Changes not staged for commit:
  modified:   src/auth.js
```

### Восстановить файл из конкретного коммита

```bash
# Восстановить версию файла из предыдущего коммита
git restore --source=HEAD~1 -- src/auth.js

# Из конкретного коммита
git restore --source=abc1234 -- src/auth.js

# Из определённой ветки
git restore --source=develop -- config.json
```

**Практический пример:**
```bash
# Вы случайно удалили важный файл
# Но помните что он был в коммите abc1234

$ git restore --source=abc1234 -- src/deleted-file.js

$ git status
On branch main
Changes to be committed:
  new file:   src/deleted-file.js

# Файл восстановлен и в staging, готов к коммиту!
```

### Комбинирование флагов

```bash
# Отменить staged И unstaged изменения (вернуть к HEAD)
git restore --staged --worktree -- src/auth.js

# Более короткий вариант
git restore -SW -- src/auth.js
```

## Сравнение с git checkout

Для пользователей старых версий Git — вот как переходить со старого синтаксиса на новый:

### Для веток

| Старый синтаксис (checkout) | Новый синтаксис (switch) |
|-----|-----|
| `git checkout main` | `git switch main` |
| `git checkout -b new-feature` | `git switch -c new-feature` |
| `git checkout --track origin/feature` | `git switch --track origin/feature` |
| `git checkout -` | `git switch -` |
| `git checkout -b feature develop` | `git switch -c feature develop` |
| `git checkout --detach abc1234` | `git switch --detach abc1234` |

### Для файлов

| Старый синтаксис (checkout) | Новый синтаксис (restore) |
|-----|-----|
| `git checkout -- src/file.js` | `git restore src/file.js` |
| `git checkout abc1234 -- file.js` | `git restore --source=abc1234 -- file.js` |
| `git checkout HEAD -- .` | `git restore .` |
| нет прямого аналога | `git restore --staged file.js` |

## Когда использовать что: принятие решения

**Используйте `git switch` когда:**
- Нужно переключиться на другую ветку
- Нужно создать новую ветку
- Нужно переключиться на удалённую ветку
- Нужно вернуться на предыдущую ветку (`git switch -`)

**Используйте `git restore` когда:**
- Нужно отменить изменения в файле(ах)
- Нужно отменить staging
- Нужно восстановить файл из старого коммита
- Нужно вернуть файл в состояние из другой ветки

**Используйте `git checkout` когда:**
- Вы на старой версии Git (< 2.23)
- Вам нужна максимальная совместимость со старыми скриптами

## FAQ: Частые вопросы про switch и restore

**Q: Работает ли git checkout всё ещё?**

A: Да, `git checkout` всё ещё работает и не планируется удаляться. Но он считается устаревшим (deprecated) и рекомендуется использовать `git switch` и `git restore` для новых проектов.

**Q: Какая версия Git мне нужна?**

A: `git switch` и `git restore` появились в Git 2.23 (август 2019). Проверьте вашу версию:
```bash
git --version
```
Если у вас более старая версия, обновите Git или используйте `git checkout`.

**Q: Что будет если я переключусь с незакоммиченными изменениями?**

A: Если изменения конфликтуют с целевой веткой, Git выдаст ошибку и не позволит переключиться. Вы должны либо закоммитить, либо отложить изменения (`git stash`), либо откатить их (`git restore`).

**Q: В чём разница между git restore и git checkout для файлов?**

A: В функциональности нет разницы — они делают то же самое. Разница в синтаксисе: `git restore` понятнее и специализирован именно для файлов, в то время как `git checkout` смешивает две разные задачи (ветки и файлы).

**Q: Может ли git switch переключать теги?**

A: Да, вы можете переключиться на тег (он создаст detached HEAD):
```bash
git switch --detach v1.2.3
```
Это переведёт вас к состоянию на момент релиза v1.2.3.

## Заключение

`git switch` и `git restore` — это современный, понятный способ работы с ветками и файлами в Git:

- **`git switch`** для всей работы с ветками (переключение, создание, отслеживание удалённых)
- **`git restore`** для восстановления и отката файлов

Эти команды имеют намного более логичный синтаксис чем старый `git checkout` и рекомендуются всем, кто использует Git 2.23 или новее.

Для более глубокого понимания работы с ветками см. {{< relref "pereklyuchitsya-mezhdu-vetkami" >}} и {{< relref "kak-sozdat-vetku-git" >}}. Для отката изменений см. {{< relref "otmenit-izmeneniya-fajl-git" >}} и {{< relref "git-stash" >}}.
