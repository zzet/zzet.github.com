---
title: "cannot pull with rebase: unstaged changes — как исправить"
description: "Ошибка cannot pull with rebase: you have unstaged changes. Решение: git stash → git pull --rebase → git stash pop. Или commit, или git restore."
date: 2025-12-05
lastmod: 2025-12-05
draft: false
slug: "git-cannot-rebase-unstaged"
keywords: ["cannot rebase you have unstaged changes", "cannot pull with rebase", "git remove unstaged changes", "error cannot pull with rebase unstaged", "git rebase unstaged changes"]
tags: ["git", "beginner", "troubleshooting"]
categories: ["git"]
---

Вы пытаетесь обновить свою ветку с помощью команды `git pull --rebase` или просто `git pull`, но получаете ошибку:

```
error: cannot pull with rebase: you have unstaged changes.
error: please commit or stash them.
```

Эта ошибка означает, что в вашей рабочей директории есть изменённые файлы, которые не добавлены в staging area (индекс) и не закоммичены. Git не позволяет выполнить rebase, потому что это может привести к конфликтам или потере данных.

**Почему это происходит?** При выполнении `git pull --rebase` Git воспроизводит ваши локальные коммиты поверх обновлённой удалённой ветки. Это требует чистого состояния рабочей директории. Если у вас есть незакоммиченные изменения, они могут конфликтовать с коммитами, которые Git пытается применить.

В этой статье вы узнаете несколько надёжных способов решить эту проблему.

## Быстрое решение: git stash

Самый простой и быстрый способ — это использовать `git stash`. Команда `stash` временно сохраняет ваши изменения, позволяя вам выполнить операцию, а затем восстановить эти изменения.

**Пошаговые инструкции:**

```bash
# 1. Сохраняем незакоммиченные изменения во временное хранилище
git stash

# 2. Выполняем pull с rebase
git pull --rebase

# 3. Восстанавливаем наши изменения
git stash pop
```

**Подробный пример:**

Допустим, вы работали над файлом `src/auth.js`:

```bash
$ git status
On branch feature/login
Your branch is behind 'origin/feature/login' by 2 commits.
  (use "git pull" to update the branch)

Changes not staged for commit:
  (use "git add <file>..." to update what will be included in what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   src/auth.js

$ git stash
Saved working directory and index state WIP on feature/login: abc1234 Add login form

$ git pull --rebase
Successfully rebased and updated refs/heads/feature/login.

$ git stash pop
On branch feature/login
Changes not staged for commit:
  (use "git add <file>..." to update what will be included in what will be committed)
  modified:   src/auth.js

no changes added to commit but untracked files present (working directory clean)
```

**Важно:** если после `git stash pop` возникнут конфликты, вам потребуется вручную их разрешить. Подробнее об этом в разделе FAQ.

## Альтернатива: сделать commit

Если ваши изменения уже достаточно значимые, вы можете просто закоммитить их перед pull:

```bash
# Добавляем все изменения в индекс
git add .

# Коммитим с сообщением (например, WIP = "Work In Progress")
git commit -m "WIP: работа над функцией логина"

# Теперь pull --rebase сработает нормально
git pull --rebase
```

После успешного pull вы можете переписать историю коммитов, если нужно. Например, если хотите объединить коммит WIP с предыдущим:

```bash
# Проверяем историю
git log --oneline -5

# Переписываем 2 последних коммита (interactive rebase)
git rebase -i HEAD~2
```

В интерактивном окне вы можете пометить второй коммит как `squash` (или `s`), чтобы объединить его с первым.

## Альтернатива: отменить изменения

Если изменения вам больше не нужны, просто откатите их:

```bash
# Откатываем все изменения в текущей директории
git restore .

# Или откатываем конкретный файл
git restore src/auth.js

# Теперь можем выполнить pull --rebase
git pull --rebase
```

**Важно:** команда `git restore` необратима — изменения будут потеряны. Используйте её только если вы уверены, что не нужны эти изменения.

## Почему rebase требует чистого состояния

Понимание различия между обычным `git pull` и `git pull --rebase` поможет вам разобраться, почему rebase требует чистого состояния.

**git pull (слияние/merge):**

```bash
git pull
# Эквивалентно:
# git fetch origin
# git merge origin/branch
```

При обычном `git pull` Git создаёт новый коммит слияния, объединяющий вашу ветку с удалённой. Merge не требует чистого состояния рабочей директории, поскольку процесс слияния может обработать изменения параллельно.

**git pull --rebase (переперед):**

```bash
git pull --rebase
# Эквивалентно:
# git fetch origin
# git rebase origin/branch
```

При `git pull --rebase` Git выполняет следующее:
1. Загружает обновления с удалённого сервера (`fetch`)
2. Берёт все ваши локальные коммиты поверх вашей текущей ветки
3. Переигрывает эти коммиты поверх обновлённой удалённой ветки

Процесс переигрывания требует чистого состояния, потому что Git применяет ваши коммиты как патчи. Если в рабочей директории есть незакоммиченные изменения, они могут конфликтовать с применением этих патчей, что приведёт к непредсказуемым результатам.

Вот визуальная иллюстрация:

```
# До rebase:
Local:   A - B - C (ваши коммиты)
Remote:  A - X - Y - Z

# После rebase (эти коммиты переигрываются):
Result:  A - X - Y - Z - B' - C' (ваши коммиты переиграны сверху)
```

Если в рабочей директории есть незакоммиченные изменения, Git не может безопасно применить эти патчи.

## Настройка pull.rebase

Если вы часто используете `--rebase` с `git pull`, можно настроить Git на использование rebase по умолчанию:

```bash
# Для текущего репозитория
git config pull.rebase true

# Для всех репозиториев (глобально)
git config --global pull.rebase true
```

Теперь `git pull` будет эквивалентен `git pull --rebase` по умолчанию.

**Проверить текущее значение:**

```bash
git config pull.rebase
# Выведет: true, false или ничего (если не установлено)
```

**Значения параметра:**

- `pull.rebase = false` (по умолчанию) — использовать merge
- `pull.rebase = true` — использовать rebase
- `pull.rebase = interactive` — использовать интерактивный rebase (редко используется)

**Обратно на merge:**

```bash
git config pull.rebase false
# или удалить настройку
git config --unset pull.rebase
```

## git rebase --autostash

Если у вас часто возникает эта проблема, решение ещё проще. Используйте флаг `--autostash`, который автоматически сохраняет и восстанавливает изменения:

```bash
# Одноразовое использование
git pull --rebase --autostash

# Или для обычного rebase
git rebase --autostash origin/main
```

**Глобальная настройка:**

```bash
git config rebase.autoStash true
# или глобально
git config --global rebase.autoStash true
```

Теперь каждый `git rebase` и `git pull --rebase` будут автоматически использовать stash, и вам не нужно выполнять три команды вручную.

**Проверить настройку:**

```bash
git config rebase.autoStash
# true или ничего (если отключено)
```

## Как найти незакоммиченные изменения

Перед тем как выполнять pull или rebase, полезно знать, что именно у вас изменилось.

**Просмотр статуса:**

```bash
git status
```

Эта команда показывает:
- Отслеживаемые файлы с изменениями (Changes not staged for commit)
- Новые неотслеживаемые файлы (Untracked files)
- Файлы в staging area (Changes to be committed)

**Просмотр различий:**

```bash
# Просмотр изменений в отслеживаемых файлах
git diff

# Просмотр изменений в staging area
git diff --staged

# Просмотр изменений конкретного файла
git diff src/auth.js
```

**Пример вывода:**

```bash
$ git diff src/auth.js
diff --git a/src/auth.js b/src/auth.js
index 1234567..abcdefg 100644
--- a/src/auth.js
+++ b/src/auth.js
@@ -10,3 +10,5 @@ function login(username, password) {
   // check credentials
+  console.log("Debug: logging in user");
+  validateUser(username, password);
```

**Перемещение изменений между статусами:**

```bash
# Добавить файл в staging area
git add src/auth.js

# Убрать файл из staging area (но не откатывать изменения)
git restore --staged src/auth.js

# Откатить все изменения в файле (вернуть на версию из HEAD)
git restore src/auth.js
```

## Типичные ошибки и их решение

**Ошибка 1: "Your branch and 'origin/main' have diverged"**

Это означает, что вы делали локальные коммиты, а удалённая ветка тоже обновилась. Решение:

```bash
git stash
git pull --rebase
git stash pop
```

**Ошибка 2: конфликты после stash pop**

Если после `git stash pop` возникли конфликты слияния:

```bash
# Разрешите конфликты в редакторе
# Затем завершите stash pop
git add .
git stash drop  # или git stash pop завершится автоматически
```

**Ошибка 3: "fatal: Not a valid object name"**

Это может произойти, если вы случайно удалили стеш. В этом случае восстановить данные невозможно из Git (но можно попробовать восстановление уровня ОС).

## FAQ: часто задаваемые вопросы

**1. В чём разница между `git pull` и `git pull --rebase` в контексте этой ошибки?**

`git pull` выполняет merge, который обычно не требует чистого состояния рабочей директории. `git pull --rebase` выполняет rebase, который требует чистого состояния. Если у вас возникла ошибка именно с rebase, значит вы используете `--rebase` или у вас настроен `pull.rebase = true`.

**2. После `git stash pop` возникли конфликты. Что делать?**

Это нормально. Git пытался переприменить ваши изменения, но они конфликтовали с текущим состоянием. Вам нужно:
1. Разрешить конфликты вручную в редакторе
2. Выполнить `git add .`
3. Конфликты разрешены, и ваш stash будет завершён

**3. Как предотвратить эту ошибку в будущем?**

Есть несколько способов:
- Всегда коммитьте ваши изменения перед pull
- Используйте `git stash` перед pull
- Настройте `rebase.autoStash = true`, чтобы автоматически сохранять изменения
- Частые коммиты помогают избежать накопления изменений

**4. Что произойдёт, если я выполню `git rebase origin/main` без pull?**

Это просто переигрывает ваши локальные коммиты поверх локальной копии origin/main. Это не обновляет удалённую информацию, поэтому сначала нужно выполнить `git fetch`, а потом `git rebase origin/main`. Или используйте `git pull --rebase`, который делает оба действия.

**5. Можно ли включить `git autostash` глобально?**

Да, используйте `git config --global rebase.autoStash true`. Тогда каждый rebase будет автоматически использовать stash, и вам не нужно думать об этом.

## Резюме

Ошибка "cannot pull with rebase: you have unstaged changes" легко решается несколькими способами:

1. **Самый простой**: `git stash` → `git pull --rebase` → `git stash pop`
2. **Альтернатива**: закоммитить изменения перед pull
3. **Если изменения не нужны**: `git restore .` перед pull
4. **Долгосрочное решение**: `git config rebase.autoStash true`

Выберите подход, который лучше всего подходит для вашего рабочего процесса. Для большинства людей рекомендуется включить `rebase.autoStash`, чтобы не задумываться об этом в будущем.

## Внутренние ссылки

Для более глубокого понимания Git рекомендуем прочитать:

- {{< relref "git-stash" >}} — полная информация о команде stash
- {{< relref "git-rebase" >}} — как работает rebase и когда его использовать
- {{< relref "git-pull" >}} — различия между merge и rebase при pull
- {{< relref "otmenit-izmeneniya-fajl-git" >}} — как откатывать изменения в Git
