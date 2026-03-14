---
title: "Как отменить merge в Git: git revert -m для merge коммитов"
description: "Как отменить merge коммит в Git. git revert для merge требует -m 1. Пошаговый пример, решение проблемы повторного слияния."
date: 2026-01-23
lastmod: 2026-03-14
draft: false
slug: "git-revert-merge-commit"
keywords: ["git revert merge commit", "git revert merge", "отменить merge git", "git revert -m", "git revert слияние"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Откат обычного коммита в Git простой — просто выполните `git revert HASH` и готово. Но merge коммиты отличаются: они имеют два родителя (parent'а) вместо одного, потому что они объединяют две разные ветки.

Когда вы пытаетесь отменить merge коммит обычным `git revert`, Git не знает какого из двух родителей считать основным. Вот почему нужен параметр `-m`:

```bash
# Неправильно для merge коммитов:
git revert abc1234
# error: commit abc1234 is a merge but no -m option was given.

# Правильно:
git revert -m 1 abc1234
```

Параметр `-m 1` означает "считай первого родителя (parent 1) правильной версией, а всё что со второго родителя считай ошибкой".

В этой статье мы разберём почему merge коммиты сложнее, как правильно их откатывать, и как решить проблему "повторного слияния" которая часто возникает после отката.

## Почему обычный git revert не работает на merge коммитах

### Структура обычного коммита

Обычный коммит имеет одного родителя (parent):

```
A (old commit)
 \
  B (new commit) ← одного родителя: A
```

Когда вы выполняете `git revert B`, Git говорит: "отмени все изменения между A и B". Всё просто.

### Структура merge коммита

Merge коммит имеет ДВА родителя:

```
    A (commit on main)
    |
    |    C (commit on feature branch)
    |   /
    | /
    B (merge commit) ← two parents: A and C
```

Когда вы выполняете `git revert B`, Git не знает:
- Отменить изменения между A и B?
- Или отменить изменения между C и B?
- Или может быть отменить все изменения которые пришли из feature ветки?

Вот почему Git требует `-m` флаг для уточнения.

### Ошибка которую вы увидите

```bash
$ git revert abc1234
error: commit abc1234 is a merge but no -m option was given.
hint: revert was unable to revert abc1234. Consider you may want one of these options:
hint:     --mainline 1
hint:     --mainline 2
```

Параметры:
- `-m 1` или `--mainline 1` — откатить к первому родителю (обычно это main/master ветка)
- `-m 2` или `--mainline 2` — откатить ко второму родителю (обычно это feature ветка)

## Как выполнить git revert -m 1 правильно

### Найдите merge коммит

Сначала найдите хеш merge коммита который хотите отменить:

```bash
# Посмотрите все коммиты на текущей ветке
git log --oneline

# Или посмотрите граф с визуализацией merge'ов
git log --oneline --graph

# Или найдите merge'ы по ключевому слову
git log --oneline | grep -i merge

# Или посмотрите только merge коммиты
git log --oneline --merges
```

**Пример:**
```bash
$ git log --oneline --graph -n 10
*   d7f8e9a Merge branch 'feature-payment'
|\
| * c4b3a2d Add payment processing
| * b1c2d3e Add payment validation
| * a1b2c3d Add payment UI
|/
* f4e5d6c Update documentation
* 8x9y0z1 Add new feature
```

Здесь `d7f8e9a` это merge коммит.

### Проверьте информацию о merge коммите

Перед откатом хорошо бы посмотреть детали merge коммита:

```bash
# Посмотрите всю информацию о коммите
git show d7f8e9a

# Или посмотрите какие файлы были изменены
git show --stat d7f8e9a

# Или посмотрите только кого был родителя
git cat-file -p d7f8e9a | grep parent
```

**Пример вывода:**
```bash
$ git show d7f8e9a --stat
commit d7f8e9a1234567890abcdef1234567890 (HEAD -> main)
Merge: c4b3a2d f4e5d6c
Author: John Doe <john@example.com>
Date:   Fri Jan 23 14:30:00 2026 +0300

    Merge branch 'feature-payment'

 src/payment.js     | 150 ++++++++++++++++++++++
 src/validation.js  |  50 ++++++++
 tests/payment.test.js | 100 ++++++++++++++++
 3 files changed, 300 insertions(+)
```

Строка `Merge: c4b3a2d f4e5d6c` показывает двух родителей:
- `c4b3a2d` — первый родитель (обычно main ветка)
- `f4e5d6c` — второй родитель (обычно feature ветка)

### Выполните git revert -m 1

```bash
# Откатить merge коммит
git revert -m 1 d7f8e9a

# Git откроет редактор для сообщения отмены
# Сообщение по умолчанию: "Revert "Merge branch 'feature-payment'""
# Вы можете оставить его или отредактировать
```

**Полный пример:**
```bash
$ git revert -m 1 d7f8e9a
[main 8x9y0z1] Revert "Merge branch 'feature-payment'"
 Date: Fri Jan 23 14:31:00 2026 +0300
 src/payment.js     | 150 -----------------------------------------------
 src/validation.js  |  50 ---------
 tests/payment.test.js | 100 ---------------------------------
 3 files changed, 300 deletions(-)

$ git status
On branch main
nothing to commit, working tree clean

# Проверьте результат
$ git log --oneline -n 3
8x9y0z1 Revert "Merge branch 'feature-payment'"
d7f8e9a Merge branch 'feature-payment'
f4e5d6c Update documentation
```

### Запушите отмену merge'а

```bash
# Запушьте новый коммит отмены
git push

# Теперь в удалённом репо есть отмена merge'а
# Все видят что произошло
```

## Проверить что откат прошёл успешно

После `git revert -m 1` важно проверить что откат выполнился правильно:

### Посмотрите граф истории

```bash
$ git log --oneline --graph -n 5
* 8x9y0z1 Revert "Merge branch 'feature-payment'" (HEAD -> main)
*   d7f8e9a Merge branch 'feature-payment'
|\
| * c4b3a2d Add payment processing
| * b1c2d3e Add payment validation
| * a1b2c3d Add payment UI
|/
```

История выглядит правильно — есть merge, потом его отмена.

### Посмотрите diff между отменённым состоянием и текущим

```bash
# Посмотрите какие файлы вернулись
git show 8x9y0z1

# Или посмотрите diff между merge и его отменой
git diff d7f8e9a 8x9y0z1
```

### Проверьте что коды работают

```bash
# Перестройте проект
npm install
npm run build

# Или для Python
pip install -r requirements.txt
python manage.py test

# Или запустите приложение и проверьте
npm start
```

## Проблема: повторное слияние той же ветки

Здесь возникает интересная и раздражающая проблема. Если вы:

1. Смержили feature ветку в main (коммит A)
2. Отменили merge (git revert -m 1)
3. Потом пытаетесь смержить feature ветку ещё раз

Git скажет что нечего мержить, потому что все коммиты из feature ветки уже в истории main'а!

### Почему это происходит

```
main:    ... → A (Merge) → B (Revert Merge) → ?
feature: ... → a → b → c

Git видит: a, b, c уже есть в истории main (в коммите A)
Поэтому: нечего мержить!
```

### Решение: git revert revert

Если вы хотите смержить ту же ветку ещё раз после отката, нужно откатить саму отмену:

```bash
# Найдите коммит отмены
git log --oneline | grep Revert
# 8x9y0z1 Revert "Merge branch 'feature-payment'"

# Отмените саму отмену (revert revert)
git revert 8x9y0z1

# Теперь можете смержить feature ветку ещё раз
git merge feature-payment
```

**Полный пример проблемы и решения:**
```bash
# Первый merge
$ git merge feature-login
# Успешно смержено

# Отмена merge
$ git revert -m 1 HEAD
# Успешно отменено

# Потом коллега просит смержить ту же ветку ещё раз
$ git merge feature-login
Already up to date.
# Неправильно! Ветка не смёрнена!

# Решение: откатить отмену
$ git revert HEAD
# Отменяем отмену merge'а

$ git merge feature-login
# Теперь работает!
```

### Альтернативное решение: rebase и cherry-pick

Если простое повторное слияние не работает, можно скопировать нужные коммиты вручную:

```bash
# Найдите коммиты которые хотите скопировать
git log --oneline feature-login..main  # что в feature но не в main

# Или используйте cherry-pick
git cherry-pick abc1234 def5678 ghi9012

# Или переделайте feature ветку
git checkout feature-login
git rebase main
```

Подробнее об этом в {{< relref "git-reset" >}}.

## Полный сценарий: production откат

Давайте посмотрим на реальный сценарий где merge отката в production:

### Ситуация

Вы смержили feature ветку в production и заметили что система сломана. Нужно откатить быстро.

```bash
# Вы на main (production)
$ git log --oneline -n 5
d7f8e9a Merge branch 'feature-new-payment' (HEAD -> main)
c4b3a2d Some other commit
b1c2d3e Previous commit
a1b2c3d Old commit
f4e5d6c Even older

# Заметили баги в коммите d7f8e9a
```

### Решение

```bash
# Шаг 1: Откатить merge коммит
$ git revert -m 1 d7f8e9a
[main 8x9y0z1] Revert "Merge branch 'feature-new-payment'"
 ...
 5 files changed, 100 deletions(-)

# Шаг 2: Проверить что откат выглядит правильно
$ git log --oneline -n 3
8x9y0z1 Revert "Merge branch 'feature-new-payment'" (HEAD -> main)
d7f8e9a Merge branch 'feature-new-payment'
c4b3a2d Some other commit

# Шаг 3: Запушить откат
$ git push

# Шаг 4: Deploy production
# CI/CD система автоматически deploy'ит changes
# Или вы делаете это вручную

$ npm run build
$ npm run deploy:production

# Production восстановлена!
```

## Разница между -m 1 и -m 2

Обычно используется `-m 1`, но когда использовать `-m 2`?

### -m 1: основная ветка (parent 1)

```bash
git revert -m 1 MERGE_HASH
```

Это отменяет всё что пришло из feature ветки (parent 2). Используйте это в 99% случаев.

### -m 2: feature ветка (parent 2)

```bash
git revert -m 2 MERGE_HASH
```

Это отменяет всё что было в основной ветке до merge. Это редко нужно.

**Пример когда нужна -m 2:**

Если вы случайно смержили в неправильное направление:

```bash
# Вы смержили main в feature (неправильно!)
$ git merge main  # Когда вы на feature

# Нужно отменить
$ git revert -m 2 HEAD  # -m 2 потому что main это second parent
```

## FAQ: Частые вопросы про revert merge коммитов

**Q: Что означает -m 1?**

A: Это флаг `--mainline 1` который говорит Git: "откати против первого родителя (обычно это main ветка)". То есть откатить всё что пришло из feature ветки.

**Q: Когда использовать -m 1 а когда -m 2?**

A: Используйте `-m 1` в 99% случаев. `-m 2` нужна только если merge был в необычном направлении (например вы смежили main в feature, а не наоборот).

**Q: Как я могу узнать кто первый родитель?**

A: Выполните:
```bash
git cat-file -p MERGE_HASH | grep parent
```
Первая строка — первый родитель, вторая — второй.

**Q: Что делать если я выполнил git revert -m 2 вместо -m 1?**

A: Ничего страшного! Просто отмените сам эту отмену:
```bash
git revert HEAD
```

**Q: Отличается ли git revert от git reset для merge коммитов?**

A: Да! `git reset --hard HEAD~1` полностью удаляет merge коммит (только если он локальный). `git revert -m 1` создаёт новый коммит отмены и сохраняет историю. Для опубликованных merge'ов всегда используйте revert.

## Заключение

Откат merge коммитов требует особого внимания из-за их двух родителей:

- Используйте `git revert -m 1 HASH` для отката merge коммита
- `-m 1` означает "откатить против первого родителя" (обычно main)
- Всегда используйте `git revert` для опубликованных merge'ов, никогда не используйте `git reset --hard`
- Будьте готовы к проблеме "повторного слияния" — решается через `git revert` самой отмены

Для более глубокого понимания откатывания см. {{< relref "git-revert" >}} и {{< relref "git-merge" >}}.
