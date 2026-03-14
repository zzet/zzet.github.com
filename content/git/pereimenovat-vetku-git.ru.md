---
title: "Как переименовать ветку в Git: локально и на GitHub/GitLab"
description: "Как переименовать ветку в Git. git branch -m новое-имя для локальной, git push origin --delete старое-имя для удалённой. Переименование master в main."
date: 2025-12-08
lastmod: 2025-12-08
draft: false
slug: "pereimenovat-vetku-git"
keywords: ["переименовать ветку git", "rename branch git", "git rename branch local", "git rename branch remote", "переименование ветки git", "как переименовать ветку git"]
tags: ["git", "beginner"]
categories: ["git"]
---

Git не содержит команды `git rename` для переименования веток. Вместо этого используется команда `git branch -m`, которая означает "move" (переместить/переименовать). Процесс переименования отличается для локальных веток и веток, уже загруженных на удалённый сервер (GitHub, GitLab, Bitbucket и т.д.).

В этой статье вы узнаете, как переименовать ветку в обоих случаях, включая специальный сценарий переименования `master` в `main`.

## Переименовать текущую ветку

Если вы находитесь на ветке, которую хотите переименовать, используйте:

```bash
git branch -m новое-имя-ветки
```

**Пример:**

```bash
# Вы находитесь на ветке "feature/auth"
git branch -m feature/authentication

# Проверяем
git branch
# * feature/authentication
#   main
#   develop
```

Эта команда переименовывает текущую ветку локально. Если ветка была загружена на удалённый сервер, нужно выполнить дополнительные шаги (см. раздел "Обновить удалённую ветку").

## Переименовать ветку, находясь на другой ветке

Если вы хотите переименовать ветку, не переключаясь на неё, укажите старое и новое имена явно:

```bash
git branch -m старое-имя новое-имя
```

**Пример:**

```bash
# Находимся на ветке main
git branch -m feature/bugfix feature/critical-bugfix

# Проверяем
git branch -a
# feature/critical-bugfix
# * main
#   develop
```

Это удобно, когда вы хотите переименовать ветку, не отвлекаясь от текущей работы.

## Обновить удалённую ветку (push + delete старого имени)

Если ветка уже загружена на GitHub, GitLab или другой удалённый сервер, нужно выполнить три дополнительных шага:

```bash
# Шаг 1: загружаем новое имя ветки на сервер
git push origin новое-имя-ветки

# Шаг 2: удаляем старое имя ветки с сервера
git push origin --delete старое-имя-ветки

# Шаг 3: обновляем upstream tracking (связь локальной ветки с удалённой)
git branch -u origin/новое-имя-ветки
```

**Подробный пример:**

```bash
# 1. Локально переименовываем ветку
git branch -m feature/auth feature/user-auth

# 2. Загружаем новое имя
git push origin feature/user-auth
# Counting objects: 100% (5/5), done.
# To github.com:user/repo.git
#  * [new branch]      feature/user-auth -> feature/user-auth

# 3. Удаляем старое имя с сервера
git push origin --delete feature/auth
# To github.com:user/repo.git
#  - [deleted]         feature/auth

# 4. Обновляем upstream tracking
git branch -u origin/feature/user-auth
# Branch 'feature/user-auth' set up to track remote branch 'feature/user-auth' from 'origin'.

# Проверяем результат
git branch -vv
# * feature/user-auth 1a2b3c4 [origin/feature/user-auth] Add user authentication
#   main              9z8y7x6 [origin/main] Update README
```

## Полный сценарий — пошаговый пример

Представим, что вы работаете над веткой `feature/auth` на GitHub. Ветка содержит несколько коммитов, и вы решили переименовать её в `feature/user-auth` для большей ясности.

**Исходное состояние:**

```bash
$ git branch -a
* feature/auth          (вы находитесь здесь)
  main
  remotes/origin/main
  remotes/origin/feature/auth
```

**Шаг 1: локально переименовываем**

```bash
git branch -m feature/user-auth
```

**Шаг 2: проверяем локальное переименование**

```bash
git branch -a
# * feature/user-auth   (локальная ветка переименована)
#   main
#   remotes/origin/main
#   remotes/origin/feature/auth  (удалённая ветка ещё со старым именем!)
```

**Шаг 3: загружаем новое имя на сервер**

```bash
git push origin feature/user-auth
# Counting objects: 100% (7/7), done.
# ...
# To github.com:yourname/repo.git
#  * [new branch]      feature/user-auth -> feature/user-auth
```

**Шаг 4: удаляем старое имя с сервера**

```bash
git push origin --delete feature/auth
# To github.com:yourname/repo.git
#  - [deleted]         feature/auth
```

**Шаг 5: обновляем upstream tracking**

```bash
git branch -u origin/feature/user-auth
# Branch 'feature/user-auth' set up to track remote branch 'feature/user-auth' from 'origin'.
```

**Финальное состояние:**

```bash
$ git branch -vv
* feature/user-auth 1a2b3c4 [origin/feature/user-auth] Add authentication logic
  main              9z8y7x6 [origin/main] Update README
```

Готово! Ветка успешно переименована локально и на удалённом сервере.

## Переименовать master в main

Переименование главной ветки проекта является частым сценарием. Это требует дополнительных шагов, чтобы все коллеги могли обновить свои локальные копии.

**Шаг 1: локально переименовываем ветку**

```bash
git branch -m master main
```

**Шаг 2: загружаем новую ветку main на сервер**

```bash
git push origin main
# Counting objects: 100% (25/25), done.
# ...
# To github.com:yourname/repo.git
#  * [new branch]      main -> main
```

**Шаг 3: удаляем старую ветку master с сервера**

```bash
git push origin --delete master
# To github.com:yourname/repo.git
#  - [deleted]         master
```

**Шаг 4: обновляем upstream tracking**

```bash
git branch -u origin/main
# Branch 'main' set up to track remote branch 'main' from 'origin'.
```

**Шаг 5: обновляем default branch на GitHub**

На GitHub (и других сервисах) нужно установить новую ветку по умолчанию:

1. Откройте репозиторий на GitHub
2. Перейдите в Settings → Branches
3. Под "Default branch" кликните на иконку изменения
4. Выберите `main` вместо `master`
5. Подтвердите изменение

**Для коллег: что нужно сделать локально**

После того как главная ветка переименована, каждый член команды должен выполнить:

```bash
# Загружаем обновлённую информацию о удалённых ветках
git fetch origin

# Удаляем локальную ссылку на старую ветку (если она существует)
git branch -d master

# Переименовываем локальную ветку master в main
git branch -m master main

# Обновляем upstream tracking
git branch -u origin/main main

# Проверяем статус
git status
# On branch main
# Your branch is up to date with 'origin/main'.
```

Или, если локальная ветка была только отслеживающей:

```bash
git fetch origin
git checkout main
git branch -d master
```

## Переименование через GitHub/GitLab UI

Если вы не любите командную строку или хотите переименовать ветку, но не хотите делать это локально, можете использовать веб-интерфейс.

**Для GitHub:**

1. Откройте репозиторий на GitHub
2. Нажмите на значок веток (иконка с ветвлением)
3. Найдите ветку, которую хотите переименовать
4. Нажмите на иконку карандаша (редактировать) рядом с её названием
5. Введите новое имя
6. Нажмите кнопку переименования

GitHub автоматически:
- Обновляет pull requests, которые связаны с этой веткой
- Обновляет branch protection rules
- Отправляет уведомление всем, кто следит за веткой

**Для GitLab:**

1. Откройте репозиторий на GitLab
2. Перейдите в Repository → Branches
3. Найдите ветку
4. Нажмите на иконку "Edit" (три точки)
5. Выберите "Rename branch"
6. Введите новое имя
7. Подтвердите

**Важно:** даже если вы переименуете ветку через веб-интерфейс, вам нужно обновить локальную копию:

```bash
git fetch origin
git branch -m старое-имя новое-имя
git branch -u origin/новое-имя-ветки
```

## git branch -m vs git branch -M

Git предоставляет два варианта флага переименования:

**`git branch -m` (lowercase m):**

```bash
git branch -m старое-имя новое-имя
```

Этот флаг отказывает в переименовании, если целевое имя уже существует. Это безопасный вариант, предотвращающий случайную перезапись.

**Пример ошибки:**

```bash
$ git branch -m feature/auth feature/login
# fatal: A branch named 'feature/login' already exists.
```

**`git branch -M` (uppercase M):**

```bash
git branch -M старое-имя новое-имя
```

Этот флаг принудительно переименовывает ветку, перезаписывая целевую ветку, если она уже существует. Используйте это с осторожностью!

**Пример перезаписи:**

```bash
$ git branch -M feature/auth feature/login
# (feature/login переписана, даже если существовала раньше)
```

**Рекомендация:** используйте `git branch -m` (lowercase) в большинстве случаев. Используйте `git branch -M` (uppercase) только если вы действительно уверены, что хотите перезаписать целевую ветку.

## Типичные ошибки при переименовании

**Ошибка 1: "error: refname refs/heads/main not found"**

Это означает, что локальная ветка с таким именем не существует. Проверьте её наличие:

```bash
git branch -a
# Убедитесь, что ветка существует
```

**Решение:** убедитесь, что вы указали правильное имя ветки.

**Ошибка 2: "A branch named 'feature/login' already exists"**

Попытка переименовать ветку на имя, которое уже существует:

```bash
git branch -m feature/auth feature/login
# fatal: A branch named 'feature/login' already exists.
```

**Решение:** используйте другое имя или удалите существующую ветку перед переименованием.

**Ошибка 3: "remote: error: refusing to delete the current branch"**

При попытке удалить старую ветку с удалённого сервера возникает ошибка:

```bash
git push origin --delete feature/auth
# remote: error: refusing to delete the current branch: refs/heads/feature/auth
```

Это означает, что `feature/auth` является ветью по умолчанию на сервере. Измените ветку по умолчанию в Settings на другую ветку (например, `main`), а затем повторите попытку.

**Ошибка 4: "Updates were rejected because the tip of your current branch is behind"**

```bash
git push origin feature/user-auth
# error: failed to push some refs to 'github.com:user/repo.git'
# hint: Updates were rejected because the tip of your current branch is behind
```

Это означает, что на удалённом сервере есть изменения, которых нет локально. Выполните:

```bash
git fetch origin
git merge origin/feature/user-auth
# или перебажитесь
git rebase origin/feature/user-auth
```

## FAQ: часто задаваемые вопросы

**1. Мою ветку защитили через branch protection rules на GitHub. Как переименовать защищённую ветку?**

Защищённые ветки нельзя удалять или переименовывать напрямую. Вам нужно:
1. Отключить защиту в Settings → Branches → редактировать правило
2. Выполнить переименование: `git branch -m старое-имя новое-имя`
3. Загрузить на сервер
4. Удалить старое имя
5. Включить защиту на новой ветке

**2. Я переименовал ветку, но мои коллеги работают на той же ветке. Что им нужно сделать?**

Ваши коллеги должны выполнить:

```bash
git fetch origin
git branch -m старое-имя новое-имя
git branch -u origin/новое-имя
```

Или, если они хотят просто переключиться на новую ветку:

```bash
git fetch origin
git checkout новое-имя
```

**3. Влияет ли переименование ветки на историю коммитов?**

Нет, абсолютно. Переименование ветки — это просто переименование ссылки на коммит. История коммитов и их SHA1 хэши остаются неизменными. Это полностью безопасная операция.

**4. Я случайно удалил старую ветку на GitHub перед загрузкой новой. Можно ли восстановить?**

Если вы случайно удалили ветку, её можно восстановить, если она ещё находится в reflog удалённого сервера. На GitHub это возможно в течение 90 дней. Обратитесь в GitHub Support или перейдите в Settings → Branches → Recently deleted и восстановите оттуда.

Локально вы можете восстановить удалённую ветку, если у вас есть локальная копия:

```bash
git push origin feature/auth
```

**5. Я хочу переименовать несколько веток одновременно. Есть ли способ автоматизировать это?**

Git не предоставляет встроенного способа для массового переименования, но вы можете написать скрипт:

```bash
#!/bin/bash
# rename-branches.sh
# Переименовать все ветки с префиксом "old" на "new"

git fetch origin

for branch in $(git branch -a | grep 'old-'); do
  old_name=$(echo $branch | sed 's/.*\///')
  new_name=$(echo $old_name | sed 's/old-/new-/')

  # Локально
  git branch -m $old_name $new_name

  # На сервере
  git push origin $new_name
  git push origin --delete $old_name
done
```

Используйте с осторожностью и тестируйте сначала!

## Резюме

Переименование веток в Git просто:

**Локальное переименование:**
```bash
git branch -m новое-имя          # текущую ветку
git branch -m старое новое       # другую ветку
```

**Обновление на удалённом сервере:**
```bash
git push origin новое-имя
git push origin --delete старое-имя
git branch -u origin/новое-имя
```

**Особый случай — переименование master в main:**
1. Переименовать локально
2. Загрузить новое имя
3. Удалить старое имя на сервере
4. Изменить default branch в GitHub/GitLab Settings
5. Коллеги должны обновить свои локальные копии

## Внутренние ссылки

Для более глубокого понимания работы с ветками рекомендуем прочитать:

- {{< relref "kak-sozdat-vetku-git" >}} — как создавать новые ветки
- {{< relref "udalennye-vetki-git" >}} — работа с удалёнными ветками
- {{< relref "pereklyuchitsya-mezhdu-vetkami" >}} — переключение между ветками
