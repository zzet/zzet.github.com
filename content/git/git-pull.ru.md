---
title: "git pull: как получить обновления из удалённого репозитория Git"
description: "Полное руководство по git pull. Что это, чем отличается от git fetch, как разрешить конфликты слияния. Примеры команд."
date: 2026-01-22
lastmod: 2026-01-22
draft: false
slug: "git-pull"
keywords: ["git pull origin master что делает", "как подтянуть изменения git", "git pull что это", "скачать обновления git"]
tags: ["git", "remote", "beginner"]
categories: ["git"]
aliases: []
---

В командной разработке постоянно происходит обмен кодом. Пока вы работали над своей задачей, коллеги закоммитили и запушили свои изменения. Команда `git pull` позволяет получить эти изменения и синхронизировать локальный репозиторий с удалённым.

## Что делает git pull

`git pull` — это комбинация двух команд:

```bash
git pull origin main

# Эквивалентно:
git fetch origin          # 1. Загрузить изменения с сервера
git merge origin/main     # 2. Слить их с текущей веткой
```

**`git fetch`** скачивает коммиты из удалённого репозитория, но не применяет их к вашей локальной ветке. Изменения хранятся в `origin/main` (удалённый tracking branch).

**`git merge`** объединяет скачанные изменения с вашей локальной веткой.

## Базовое использование

```bash
# Получить изменения из origin в текущую ветку
git pull

# Явно указать remote и ветку
git pull origin main
git pull origin develop
```

## git pull vs git fetch: что выбрать

Это важное различие:

| Команда | Скачивает | Применяет к локальной ветке |
|---------|-----------|------------------------------|
| `git fetch` | ✓ | ✗ |
| `git pull` | ✓ | ✓ (через merge или rebase) |

Некоторые разработчики предпочитают делать `git fetch` + просмотр + `git merge` вместо `git pull`:

```bash
git fetch origin

# Посмотреть что пришло
git log --oneline HEAD..origin/main   # Что пришло с сервера
git diff HEAD origin/main             # Разница

# Только после проверки — слить
git merge origin/main
```

Это даёт больший контроль, но требует дополнительных шагов. Для большинства случаев `git pull` удобнее.

## Флаги и опции

### --rebase: вместо merge делать rebase

```bash
git pull --rebase origin main
```

Вместо создания merge-коммита, ваши локальные коммиты «перекладываются» поверх скачанных:

```
Без --rebase:   A ← B ← C ← M (merge commit)
                         ↗
                    D ← E (ваши коммиты)

С --rebase:     A ← B ← D ← E (ваши коммиты поверх)
```

История становится линейной. Это удобно для feature-веток, где важна чистота истории.

Можно настроить rebase по умолчанию:
```bash
git config --global pull.rebase true
```

### --no-commit: слить, но не коммитить автоматически

```bash
git pull --no-commit origin main
# Изменения слиты, но merge commit ещё не создан
# Можно проверить и затем:
git commit
```

### --no-ff: всегда создавать merge commit

```bash
git pull --no-ff origin main
# Даже при fast-forward будет создан merge commit
```

## Разрешение конфликтов при pull

Конфликт возникает, если вы и коллега изменили одни и те же строки в одном файле:

```bash
git pull origin main
# Auto-merging app.js
# CONFLICT (content): Merge conflict in app.js
# Automatic merge failed; fix conflicts and then commit the result.
```

Что делать:

```bash
# 1. Посмотреть конфликтующие файлы
git status
# both modified: app.js

# 2. Открыть файл и найти маркеры конфликта
# <<<<<<< HEAD
# ваш код
# =======
# код из origin
# >>>>>>> origin/main

# 3. Отредактировать файл, оставив нужную версию

# 4. Отметить конфликт как решённый
git add app.js

# 5. Завершить merge commit
git commit
```

### Отмена pull при конфликтах

```bash
git merge --abort
# Возвращает состояние до git pull
```

## Сохранение незакоммиченных изменений перед pull

Если у вас есть незакоммиченные изменения, а нужно сделать pull:

```bash
# Способ 1: stash
git stash
git pull origin main
git stash pop       # Восстановить изменения
# Разрешить возможные конфликты stash

# Способ 2: закоммитить
git add .
git commit -m "WIP: сохранение перед pull"
git pull origin main
```

## Проверка что придёт с сервера (до pull)

```bash
# Обновить информацию об удалённых ветках без слияния
git fetch origin

# Посмотреть что пришло с сервера (ещё не у нас)
git log --oneline HEAD..origin/main

# Посмотреть наши коммиты которых нет на сервере
git log --oneline origin/main..HEAD

# Посмотреть разницу
git diff origin/main
```

## Типичные сценарии

### Обновление main в начале рабочего дня

```bash
git switch main
git pull origin main
# Теперь main актуален
git switch feature/my-task
# Продолжаем работу
```

### Обновление feature-ветки от main

```bash
git switch feature/my-task
git pull origin main  # Получить обновления main в feature-ветку
# Разрешить возможные конфликты
```

Или через rebase:
```bash
git switch feature/my-task
git pull --rebase origin main
```

### Простой pull в текущую ветку

```bash
# Если отслеживание настроено
git pull
```

## Практический пример

```bash
# Начало рабочего дня
git switch main
git pull origin main
# Already up to date. или список обновлённых файлов

git log --oneline -5
# Посмотреть что появилось нового

# Начать работу над задачей
git switch -c feature/dashboard
echo "dashboard code" > dashboard.js
git add dashboard.js
git commit -m "feat: add dashboard skeleton"

# Конец дня: обновить от main (могли прийти изменения)
git pull --rebase origin main

# Отправить работу
git push -u origin feature/dashboard
```

## Часто задаваемые вопросы

**Чем отличается `git pull` от `git fetch`?** `git fetch` только скачивает данные с сервера, не изменяя вашу рабочую директорию. `git pull` — это `git fetch` + `git merge` (или `git rebase`). `git fetch` безопаснее для проверки: можно посмотреть что пришло, прежде чем сливать.

**Что происходит при конфликте слияния?** Git останавливается и маркирует конфликтующие строки в файлах. Вы редактируете файлы вручную, затем `git add` + `git commit` для завершения.

**Когда использовать `git pull --rebase`?** Когда хотите получить линейную историю и избежать merge-коммитов. Особенно удобно для feature-веток. Некоторые команды требуют `--rebase` как стандарт.

**Как отменить неудачный pull?** Если возникли конфликты и вы хотите отменить — `git merge --abort`. Если pull уже завершён и хотите отменить — `git reset --hard HEAD~1` (осторожно!).

**Безопасно ли выполнять `git pull` с незакоммиченными изменениями?** Зависит от ситуации. Если изменения не конфликтуют — всё пройдёт хорошо. Если конфликтуют — Git откажет. Лучше закоммитить или сохранить через `git stash` перед pull.

**Почему некоторые избегают `git pull` и предпочитают `git fetch`?** `git pull` автоматически сливает изменения, что может стать неожиданностью. `git fetch` + ручной `git merge` даёт больший контроль. Оба подхода рабочие.

## Заключение

`git pull` — обязательная команда для командной работы. Выполняйте её в начале каждого рабочего дня и перед тем, как начать новую задачу — чтобы работать с актуальной версией кода.

Для линейной истории используйте `git pull --rebase`. При конфликтах — не паникуйте: `git status` покажет проблемные файлы, редактируете, `git add`, `git commit`.

Связанные команды: «[git push: отправить код на сервер]({{< relref "git-push" >}})» и «[git merge: объединение веток]({{< relref "git-merge" >}})».
