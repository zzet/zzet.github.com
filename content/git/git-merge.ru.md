---
title: "git merge: полное руководство по объединению веток"
description: "Научитесь использовать git merge для объединения веток. Fast-forward, three-way merge, конфликты, опции, примеры команд."
date: 2026-01-20
lastmod: 2026-01-20
draft: false
slug: "git-merge"
keywords: ["git как сделать merge", "git merge bash", "слияние веток git", "git merge команда", "как объединить ветки git", "git merge --no-ff", "git merge fast-forward"]
tags: ["git", "branches", "merge", "intermediate"]
categories: ["git"]
aliases: []
---

`git merge` — команда для объединения изменений из одной ветки в другую. Это центральная операция при командной разработке: когда функция готова и протестирована в отдельной ветке, она вливается в основную через merge. В этой статье разберём, как работает слияние, какие бывают типы merge и как использовать основные опции.

## Что такое merge и когда он нужен

Merge объединяет историю двух веток. Типичный сценарий:

```
main:     A ← B ← C
feature:  A ← B ← D ← E
```

После `git merge feature` на `main`:
```
main:     A ← B ← C ← M (merge commit)
                  ↗
feature:  A ← B ← D ← E
```

Merge нужен когда:
- Закончена работа над функцией, нужно влить её в `main` или `develop`
- Нужно обновить feature-ветку последними изменениями из `main`
- Pull Request / Merge Request принят на GitHub/GitLab

## Базовое использование

```bash
# 1. Переключиться на ветку, В которую сливаем
git switch main

# 2. Слить другую ветку
git merge feature/auth

# 3. Проверить результат
git log --oneline --graph
```

## Fast-forward merge

Простейший тип merge — когда основная ветка не ушла вперёд от точки ветвления:

```
main:     A ← B
feature:  A ← B ← C ← D
```

В этом случае Git просто перемещает указатель `main` на последний коммит `feature`:

```
main/feature:  A ← B ← C ← D
```

Новый коммит слияния не создаётся. История остаётся линейной.

```bash
git switch main
git merge feature/simple-fix
# Fast-forward
# index.js | 2 +-
# 1 file changed, 1 insertion(+), 1 deletion(-)
```

## Three-way merge

Когда обе ветки имеют новые коммиты после точки расхождения:

```
main:     A ← B ← C
feature:  A ← B ← D ← E
```

Git находит общего предка (`B`), сравнивает изменения в обеих ветках и создаёт новый **merge commit**:

```
main:     A ← B ← C ← M
                   ↗
feature:  A ← B ← D ← E
```

```bash
git switch main
git merge feature/complex-feature
# Merge made by the 'ort' strategy.
# auth.js | 25 +++++++++++++++++++++++++
# 1 file changed, 25 insertions(+)
```

После three-way merge в `git log` появится коммит слияния с двумя родителями.

## Опции git merge

### --no-ff: всегда создавать merge commit

Даже если возможен fast-forward, создать явный merge commit:

```bash
git merge --no-ff feature/auth
```

Это полезно для сохранения информации о том, что функция разрабатывалась в отдельной ветке:

```
Без --no-ff:   A ← B ← C ← D ← E  (история линейная)
С --no-ff:     A ← B ← C ← M
                        ↗
                   D ← E
```

Многие команды используют `--no-ff` для всех merge в `main`/`develop` — чтобы явно видеть в истории, когда вливались функции.

### --squash: собрать все коммиты в один

Объединить все коммиты из ветки в один «сжатый» коммит:

```bash
git merge --squash feature/auth
git commit -m "feat: добавлена авторизация"
```

Это очищает историю: вместо 10 «WIP» коммитов из feature-ветки появляется один аккуратный. Сама ветка при этом не сливается — нужно создать коммит вручную.

### --no-commit: слить, но не коммитить

Выполнить слияние, но остановиться перед автоматическим созданием коммита:

```bash
git merge --no-commit feature/auth
# Проверьте результат
git diff --cached
# Затем закоммитьте
git commit -m "Merge feature/auth into main"
```

Полезно для проверки результата слияния перед финализацией.

### --abort: отменить слияние в процессе

Если что-то пошло не так во время merge (например, слишком много конфликтов):

```bash
git merge --abort
# Состояние возвращается к тому, что было до merge
```

### Стратегии слияния

```bash
# Стратегия "ours": при конфликтах оставлять нашу версию
git merge -X ours feature/experimental

# Стратегия "theirs": при конфликтах брать версию из сливаемой ветки
git merge -X theirs feature/experimental
```

## Слияние с конфликтами

Конфликт возникает, когда обе ветки изменили одни и те же строки в одном файле. Git не может решить автоматически — нужно ваше вмешательство:

```bash
git merge feature/auth
# Auto-merging app.js
# CONFLICT (content): Merge conflict in app.js
# Automatic merge failed; fix conflicts and then commit the result.
```

Откройте конфликтующий файл:

```javascript
<<<<<<< HEAD
function login() {
  return "login v1";
}
=======
function login() {
  return "login v2 with OAuth";
}
>>>>>>> feature/auth
```

Разделы:
- `<<<<<<< HEAD` до `=======` — ваш текущий код (в main)
- `=======` до `>>>>>>>` — код из сливаемой ветки

Отредактируйте файл, оставив нужную версию (или объединив обе):

```javascript
function login() {
  return "login v2 with OAuth";
}
```

Затем завершите merge:
```bash
git add app.js
git commit
# Git предложит сообщение по умолчанию
```

## Отмена выполненного merge

Если merge уже создал коммит, но вы хотите отменить:

```bash
# Отменить последний коммит (merge commit)
git reset --hard HEAD~1
```

Это безопасно, если вы ещё не запушили изменения.

Если уже запушили — используйте `git revert`:
```bash
git revert -m 1 HEAD
# Создаст новый коммит, отменяющий merge
```

## Просмотр перед merge

Полезно проверить, что изменится:

```bash
# Посмотреть, какие коммиты войдут
git log main..feature/auth --oneline

# Посмотреть diff
git diff main feature/auth

# Найти общего предка (база merge)
git merge-base main feature/auth
```

## Практический пример: типичный workflow

```bash
# 1. Начало работы над задачей
git switch main
git pull origin main
git switch -c feature/user-notifications

# 2. Разработка и коммиты
git add .
git commit -m "feat: add notification service"
git add .
git commit -m "feat: add notification templates"

# 3. Обновление от main (если он ушёл вперёд)
git switch main
git pull origin main
git switch feature/user-notifications
git merge main  # Вливаем обновления main в нашу ветку

# 4. Завершение и слияние в main
git switch main
git merge --no-ff feature/user-notifications -m "Merge feature/user-notifications"

# 5. Отправка
git push origin main

# 6. Удаление ветки
git branch -d feature/user-notifications
git push origin --delete feature/user-notifications
```

## Merge и pull

`git pull` — это по сути `git fetch` + `git merge`. Когда вы выполняете `git pull origin main`, Git сначала загружает изменения (`fetch`), затем сливает их с вашей локальной веткой (`merge`).

```bash
# Эти две команды эквивалентны:
git pull origin main

git fetch origin
git merge origin/main
```

## Часто задаваемые вопросы

**Какая разница между merge и rebase?** Merge создаёт коммит слияния и сохраняет историю «как есть» — с разветвлениями. Rebase переписывает историю, делая её линейной. Merge безопаснее для общих веток, rebase — для личных feature-веток.

**Что делать при конфликтах?** Откройте конфликтующие файлы (показаны в `git status`), найдите маркеры `<<<<<<<`, `=======`, `>>>>>>>`, отредактируйте файл, затем `git add` и `git commit`.

**Когда использовать `--no-ff`?** Когда важно видеть в истории, что функция разрабатывалась отдельно. Многие команды требуют `--no-ff` для merge в основные ветки.

**Можно ли отменить merge после push?** Да, через `git revert -m 1 HEAD` — это создаст новый коммит, отменяющий merge. Не нужно переписывать историю.

**Нужно ли создавать merge commit или лучше fast-forward?** Зависит от соглашений команды. Fast-forward даёт чистую линейную историю. `--no-ff` сохраняет информацию о feature-ветках. GitHub по умолчанию предлагает оба варианта.

## Заключение

`git merge` — фундаментальный инструмент для командной разработки. Понимание разницы между fast-forward и three-way merge, умение работать с конфликтами и знание опций `--no-ff` и `--squash` — всё это делает вас эффективнее при работе с Git.

При слиянии всегда проверяйте результат через `git log --graph` — это позволяет видеть структуру истории и убеждаться, что всё слилось правильно.

## По теме

- [Конфликты при merge]({{< relref "konflikty-git-merge" >}})
- [git merge vs rebase]({{< relref "git-merge-vs-rebase" >}})
- [Отмена merge]({{< relref "git-merge-abort" >}})
- [Сообщение merge-коммита]({{< relref "git-merge-commit-message" >}})
- [Откат merge-коммита]({{< relref "git-revert-merge-commit" >}})
- [git squash]({{< relref "git-squash" >}})
- [Сравнение коммитов в GitLab]({{< relref "gitlab-compare-commits" >}})
