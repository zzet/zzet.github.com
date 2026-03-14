---
title: "git squash: как объединить несколько коммитов в один"
description: "Как сделать squash коммитов в Git. git rebase -i HEAD~N → squash/fixup. git merge --squash для ветки. Squash and merge в GitHub. Когда и зачем нужен squash."
date: 2025-12-19
lastmod: 2025-12-19
draft: false
slug: "git-squash"
keywords: ["git squash", "git squash commits", "git объединить коммиты в один", "squash and merge что это", "git squash это", "как объединить коммиты git", "git squash all commits in branch"]
tags: ["git", "intermediate"]
categories: ["git"]
---

В процессе разработки функции вы можете создать множество промежуточных коммитов: "WIP: начало работы", "fix: опечатка в коде", "refactor: переделал логику", и так далее. Когда пришло время объединить эту функцию в основную ветку, наличие десяти мелких коммитов может засорить историю проекта.

**Squash** — это процесс объединения нескольких коммитов в один. Это особенно полезно при:

- Создании pull request с чистой историей для код-ревью
- Подготовке к слиянию в основную ветку (main/master)
- Удалении промежуточных "WIP" и "fix" коммитов
- Создании красивой, читаемой истории проекта

Вот визуально, как выглядит squash:

```
До squash:
o — fix: typo in login
|
o — refactor: optimize auth logic
|
o — WIP: add password validation
|
o — main branch (merge-base)

После squash:
o — feat: add login with password validation
|
o — main branch
```

## Метод 1: интерактивный rebase — основной способ

Интерактивный rebase (`git rebase -i`) — это самый популярный способ squash'а коммитов. Он даёт вам полный контроль над процессом.

### Базовая команда

```bash
# Squash последних N коммитов
git rebase -i HEAD~N

# Пример: squash последних 4 коммитов
git rebase -i HEAD~4
```

### Пошаговый процесс

1. **Выполните команду**

```bash
git rebase -i HEAD~4
```

2. **Git откроет текстовый редактор** с таким содержимым:

```
pick abc1234 WIP: add password validation
pick def5678 refactor: optimize auth logic
pick ghi9012 fix: typo in login
pick jkl3456 refactor: improve error messages

# Rebase abc1234..jkl3456 onto abc1234 (4 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# ...
```

3. **Отредактируйте команды для коммитов**, которые хотите объединить:

```
pick abc1234 WIP: add password validation
s def5678 refactor: optimize auth logic
s ghi9012 fix: typo in login
s jkl3456 refactor: improve error messages
```

Буква `s` означает "squash" — объединить этот коммит с предыдущим, сохранив сообщение от предыдущего.

4. **Сохраните файл** и закройте редактор (Ctrl+X в nano, `:wq` в vim).

5. **Git попросит вас написать новое сообщение коммита** для объединённого коммита:

```
# Это будет новое сообщение коммита
# Вы можете использовать сообщения из всех объединённых коммитов

feat: add password validation and error handling

- Add password validation logic
- Optimize authentication logic
- Improve error messages
```

6. **Сохраните сообщение** и готово!

### Все команды интерактивного rebase

| Команда | Сокращение | Что делает |
|---------|-----------|-----------|
| pick | p | Использовать коммит как есть |
| reword | r | Использовать коммит, но изменить сообщение |
| edit | e | Использовать коммит, но остановиться для редактирования |
| squash | s | Объединить с предыдущим коммитом, сохранив оба сообщения |
| fixup | f | Объединить с предыдущим, но отбросить сообщение этого коммита |
| drop | d | Удалить коммит полностью |

### Полный пример с несколькими операциями

```bash
# Предположим, у вас есть такая история:
# abc1234 — WIP: initial setup
# def5678 — fix: bug in config
# ghi9012 — refactor: cleanup
# jkl3456 — feat: new feature
# mno7890 — WIP: needs testing

# Вы запускаете:
git rebase -i HEAD~5

# Редактируете так:
s abc1234 WIP: initial setup
f def5678 fix: bug in config
r ghi9012 refactor: cleanup
p jkl3456 feat: new feature
d mno7890 WIP: needs testing

# Результат:
# 1. abc1234 и def5678 объединены в один коммит (сообщение от abc1234)
# 2. ghi9012 остаётся, но вы сможете переписать его сообщение
# 3. jkl3456 остаётся как есть
# 4. mno7890 удаляется полностью
```

## Разница squash vs fixup

Обе команды объединяют коммиты, но по-разному обрабатывают сообщения:

### squash (s) — сохраняет оба сообщения

Когда вы используете `squash`:

```
pick abc1234 WIP: add validation
s def5678 fix: typo in message

# Git откроет редактор с обоими сообщениями:
# WIP: add validation
# fix: typo in message

# Вы можете отредактировать оба, оставить оба или выбрать нужное
```

### fixup (f) — отбрасывает сообщение

Когда вы используете `fixup`:

```
pick abc1234 feat: add validation
f def5678 fix: typo in message

# Git автоматически объединит в один коммит с сообщением:
# feat: add validation

# Сообщение "fix: typo in message" будет отброшено без запроса
```

**Когда что использовать:**

- **squash** — когда оба сообщения полезны и их нужно объединить в одно
- **fixup** — когда текущий коммит — просто исправление предыдущего ("fix typo", "wip", "debug"), и его сообщение не нужно

## Squash всей ветки до точки ветвления

Иногда нужно объединить все коммиты в ветке, начиная с точки, где она отделилась от основной ветки:

```bash
# Найти точку ветвления и squash всё от неё
git rebase -i $(git merge-base HEAD main)

# Команда git merge-base находит общий предок текущей ветки и main
# Результат: все коммиты после ветвления будут доступны для squash
```

### Практический пример

```bash
# Вы работаете в ветке feature/login
# История выглядит так:
# main: abc1234
#       |
#       o— (branch point)
#           |
#           o— def5678 (WIP: setup)
#           |
#           o— ghi9012 (add login form)
#           |
#           o— jkl3456 (fix: validation)

# Вы хотите squash'ить все коммиты в ветке в один

git rebase -i $(git merge-base HEAD main)

# Git покажет вам все 3 коммита: def5678, ghi9012, jkl3456
# Вы отмечаете первый как pick, остальные как s (squash)
# Результат: один коммит со всеми изменениями
```

## Метод 2: git merge --squash

Если вы хотите объединить всю ветку в одну без создания merge commit'а, используйте `git merge --squash`:

```bash
# Переключиться на основную ветку
git switch main

# Объединить все изменения из ветки в staging area (без коммита)
git merge --squash feature/login

# Появится сообщение:
# Squash commit -- not updating HEAD
# SQUASHING 'feature/login' into current branch...
# Automatic merge went well; stopped before committing as requested

# Теперь создайте один коммит с объединёнными изменениями
git commit -m "feat: add login functionality"
```

### Важное отличие от обычного merge

**Обычный merge:**

```bash
git switch main
git merge feature/login
# Git создаёт merge commit, сохраняя всю историю ветки
```

**git merge --squash:**

```bash
git switch main
git merge --squash feature/login
git commit -m "..."
# Все коммиты из ветки объединяются в один новый коммит
# История ветки не сохраняется в истории main
```

### Когда использовать git merge --squash

- Когда вы хотите включить все изменения из ветки, но не нужна её история
- Когда в ветке много WIP и вспомогательных коммитов
- Когда нужна чистая, линейная история в основной ветке

## Метод 3: git reset --soft (самый простой)

Если вы работаете с локальной веткой и не пушили её на сервер, `git reset --soft` — это самый быстрый способ squash'а:

```bash
# Откатить все коммиты, но оставить изменения в staging area
git reset --soft main

# Теперь все изменения в staging area, и вы можете создать один коммит
git commit -m "feat: complete feature implementation"
```

### Как это работает

```
До reset:
o— abc1234 (WIP setup)
|
o— def5678 (add logic)
|
o— ghi9012 (fix bugs)
|
o— main ← HEAD здесь

После git reset --soft main:
o— main ← HEAD здесь
|
Все изменения в staging area, можно закоммитить одним коммитом
```

### Практический пример

```bash
# Ваша история
git log --oneline
# abc1234 WIP: setup auth module
# def5678 add login logic
# ghi9012 fix password validation
# jkl3456 (HEAD -> main)

# Вы хотите объединить последние 3 коммита
git reset --soft HEAD~3

# Теперь
git status
# Все изменения в staging area

git commit -m "feat: complete authentication module"
```

## Squash and Merge в GitHub/GitLab UI

При работе с pull request'ами в GitHub и GitLab есть встроенная опция squash merge. Это удобнее, чем делать rebase локально.

### GitHub

1. Перейдите на страницу pull request
2. Нажмите кнопку **Merge pull request** (или на стрелку рядом с ней)
3. Выберите **Squash and merge**
4. Напишите сообщение коммита (GitHub автоматически предлагает объединённое сообщение)
5. Нажмите **Confirm squash and merge**

### GitLab

1. Перейдите на страницу merge request
2. В разделе **Squash commits before merge** выберите опцию **Squash commits**
3. Нажмите **Merge**

### Три варианта слияния веток

| Опция | Что делает | История |
|-------|-----------|---------|
| Create a merge commit | Создаёт merge commit, сохраняя историю ветки | Нелинейная, сохранена полная история |
| Squash and merge | Объединяет все коммиты в один, создаёт регулярный коммит | Линейная, история ветки потеряна |
| Rebase and merge | Переносит коммиты ветки на top основной ветки | Линейная, история ветки сохранена |

**Рекомендация:** используйте **squash and merge** для feature ветвей с множеством промежуточных коммитов, **rebase and merge** для чистых ветвей, **merge commit** если нужна полная история слияния.

## Force push после squash (rebase)

После интерактивного rebase история вашей локальной ветки изменилась. Если вы уже пушили эту ветку на сервер, вам потребуется force push:

```bash
# После git rebase -i ваша локальная история отличается от remote
# Проверьте изменения:
git log origin/feature-branch..HEAD

# Если всё в порядке, force push:
git push --force-with-lease origin feature-branch

# --force-with-lease безопаснее чем --force
# Она отменяется, если кто-то изменил ветку на сервере
```

### Осторожно с force push!

- Никогда не делайте force push в `main` или `master`
- Используйте `--force-with-lease` вместо `--force`
- Предупредите команду, если вы переписываете историю общей ветки
- На ветке, в которой только вы работаете, можно безопасно делать force push

## Когда НЕ нужен squash

Не всегда squash — это хорошая идея. Иногда нужно сохранить полную историю коммитов:

- **Для bisect** — команда `git bisect` использует историю коммитов для поиска нарушений, и каждый коммит должен быть отдельным для правильной работы
- **Для git blame** — детальная история коммитов помогает найти, когда и кем была добавлена строка кода
- **Когда каждый коммит значимый** — в некоторых проектах каждый коммит должен быть атомарным и готовым к продакшену
- **Публичные ветки** — если ветку используют другие разработчики, переписывание истории неудобно

## Часто задаваемые вопросы (FAQ)

**В: Как выбрать между squash, merge и rebase для слияния pull request?**

О: Руководство выбора:
- **Squash and merge** — если ветка содержит много WIP и вспомогательных коммитов. Результат: чистая, линейная история.
- **Rebase and merge** — если коммиты в ветке хорошо структурированы и каждый имеет смысл. Результат: линейная история с сохранением коммитов.
- **Create merge commit** — если нужно сохранить информацию о том, что ветка существовала. Результат: нелинейная история, но видно слияние.

**В: Как объединить все коммиты в ветке в один?**

О: Три способа:
```bash
# Способ 1: rebase с merge-base
git rebase -i $(git merge-base HEAD main)

# Способ 2: merge --squash
git switch main && git merge --squash feature && git commit

# Способ 3: reset --soft
git reset --soft main && git commit
```

**В: Я squash'ил коммиты локально и пушил их. Что теперь?**

О: Если вы изменили историю локально и теперь она отличается от удалённой ветки, используйте force push:
```bash
git push --force-with-lease origin branch-name
```

**В: В чём разница между fixup и squash?**

О:
- **squash (s)** — объединяет коммиты и позволяет отредактировать оба сообщения
- **fixup (f)** — объединяет коммиты и отбрасывает сообщение второго коммита

Используйте fixup для коммитов вроде "fix: typo" или "WIP", которые не нужны в финальной истории.

**В: Я уже смерджил ветку с squash в main. Теперь на локальной машине старая версия ветки. Что делать?**

О: После squash and merge в GitHub сама ветка не удаляется автоматически (в старых версиях Git). Удалите её локально:
```bash
git branch -d feature-branch
# или
git branch -D feature-branch (если Git не позволяет удалить)

# Удалите на удалённом сервере
git push origin --delete feature-branch
```

## Внутренние ссылки

Изучите связанные темы:
- {{< relref "git-rebase" >}} — полное руководство по интерактивному rebase
- {{< relref "git-merge-vs-rebase" >}} — сравнение разных стратегий слияния
- {{< relref "git-push-force" >}} — безопасный force push и его риски

## Заключение

Squash — это мощный инструмент для поддержания чистой и читаемой истории коммитов. Используйте его для объединения промежуточных коммитов перед слиянием в основную ветку. Помните, что интерактивный rebase даёт полный контроль, но требует внимания, в то время как `git merge --squash` проще для больших ветвей. Выбирайте метод в зависимости от вашего рабочего процесса и требований проекта.
