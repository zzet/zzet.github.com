---
title: "git apply и git patch: создание и применение патчей в Git"
description: "git apply применяет patch-файл к репозиторию. git am — применяет с сохранением коммита. Создание патчей через git diff и git format-patch."
date: 2025-12-15
lastmod: 2025-12-15
draft: false
slug: "git-apply"
keywords: ["git apply", "git patch apply", "apply patch git", "git diff apply", "git patch что это", "как применить патч git", "git format-patch"]
tags: ["git", "intermediate"]
categories: ["git"]
---

В мире разработки программного обеспечения иногда нужно передать изменения в коде без прямого доступа к репозиторию. Здесь на помощь приходят патчи — текстовые файлы, которые описывают все изменения в специальном формате diff. Патчи широко используются в Linux-сообществе, особенно в проектах, работающих через списки рассылки.

**Когда используют патчи вместо pull request:**

- **Отсутствие доступа к репозиторию** — когда разработчик не имеет прав на пуш в основной репозиторий
- **Работа через электронную почту** — в больших проектах с открытым исходным кодом (ядро Linux, Git сам по себе)
- **Безопасность** — передача критических патчей безопасности в закрытых корпоративных системах
- **Простота обмена** — можно отправить одним письмом, приложить к задаче в Jira, загрузить на форум
- **Закрытые экосистемы** — когда доступ в интернет ограничен, но есть email

Патч — это просто текстовый файл с описанием различий между двумя версиями кода. Git предоставляет несколько команд для создания и применения патчей, и мы разберём их все.

## Создание патча командой git diff

Самый простой способ создать патч — использовать команду `git diff`. Эта команда показывает различия между версиями файлов в формате, который затем можно применить с помощью `git apply`.

### Создать патч из незафиксированных изменений

Если у вас есть изменения в рабочей директории, которые вы ещё не закоммитили, создайте патч следующим образом:

```bash
# Создать патч из всех незафиксированных изменений
git diff > changes.patch

# Создать патч только для staging area (подготовленные изменения)
git diff --cached > staged-changes.patch

# Создать патч для конкретного файла
git diff -- src/main.c > main-changes.patch
```

### Создать патч из одного коммита

Чтобы создать патч из последнего коммита:

```bash
# Создать патч из последнего коммита
git diff HEAD~1 > last-commit.patch

# Создать патч из коммита, на три шага назад
git diff HEAD~3 > three-commits-ago.patch

# Создать патч из конкретного коммита (от его родителя до самого коммита)
git diff abc1234~1..abc1234 > specific-commit.patch
```

### Создать патч из диапазона коммитов

Чтобы создать патч из нескольких коммитов между ветками:

```bash
# Создать патч между ветками
git diff branch1..branch2 > feature.patch

# Создать патч между веткой и тегом
git diff v1.0..v1.1 > release.patch

# Патч от точки ветвления до текущей ветки
git diff main... > all-feature-changes.patch
```

### Что находится внутри патч-файла

Вот типичный пример содержимого патч-файла (создан командой `git diff`):

```diff
diff --git a/src/auth.js b/src/auth.js
index 1a2b3c..4d5e6f 100644
--- a/src/auth.js
+++ b/src/auth.js
@@ -10,6 +10,15 @@ function login(username, password) {
   if (!username || !password) {
     return false;
   }
+
+  // Проверка на специальные символы
+  if (!/^[a-zA-Z0-9_]+$/.test(username)) {
+    console.error('Invalid username format');
+    return false;
+  }
+
   const user = findUser(username);
   if (!user || !verifyPassword(password, user.hash)) {
     return false;
diff --git a/test/auth.test.js b/test/auth.test.js
index 9a8b7c..0e1f2g 100644
--- a/test/auth.test.js
+++ b/test/auth.test.js
@@ -15,6 +15,12 @@ describe('Login', () => {
     assert(login('user', 'password') === true);
   });

+  it('should reject username with special characters', () => {
+    assert(login('user@domain', 'password') === false);
+    assert(login('user#123', 'password') === false);
+    assert(login('user 123', 'password') === false);
+  });
+
   it('should reject wrong password', () => {
     assert(login('user', 'wrong') === false);
   });
```

Каждая секция начинается с `@@` и показывает начальную строку и количество изменённых строк в обоих файлах.

## Создание патча командой git format-patch

Команда `git format-patch` создаёт патч в специальном формате электронной почты с сохранением всей информации о коммите: автора, даты, сообщения, подписи. Это основной инструмент для работы с патчами в Linux-проектах.

### Создать патч из одного коммита

```bash
# Создать патч из последнего коммита в виде отдельного файла
git format-patch HEAD~1

# Результат: 0001-Commit-message.patch
```

### Создать патчи из нескольких коммитов

```bash
# Создать 3 отдельных файла патчей (последние 3 коммита)
git format-patch HEAD~3

# Результат:
# 0001-First-commit-message.patch
# 0002-Second-commit-message.patch
# 0003-Third-commit-message.patch

# Создать патчи в указанной директории
git format-patch HEAD~3 -o patches/

# Создать патчи, начиная с определённого коммита
git format-patch abc1234~1..abc1234
```

### Содержимое патча format-patch

Вот как выглядит патч, созданный `git format-patch`:

```email
From: Ivan Petrov <ivan.petrov@company.com>
Date: Mon, 24 Mar 2026 14:32:15 +0300
Subject: [PATCH] feat: add username validation to login function

This commit adds input validation to prevent login attempts with
invalid usernames containing special characters.

Fixes #245

---
 src/auth.js       | 9 +++++++++
 test/auth.test.js | 6 ++++++
 2 files changed, 15 insertions(+)

diff --git a/src/auth.js b/src/auth.js
index 1a2b3c..4d5e6f 100644
--- a/src/auth.js
+++ b/src/auth.js
@@ -10,6 +10,15 @@ function login(username, password) {
   if (!username || !password) {
     return false;
   }
+
+  // Проверка на специальные символы
+  if (!/^[a-zA-Z0-9_]+$/.test(username)) {
+    console.error('Invalid username format');
+    return false;
+  }
+
   const user = findUser(username);
   if (!user || !verifyPassword(password, user.hash)) {
     return false;
```

Ключевое отличие: в заголовке письма содержится информация об авторе, дате и полное сообщение коммита. Это позволяет `git am` восстановить коммит со всеми метаданными.

## Применение патча: git apply

Команда `git apply` читает патч-файл и применяет изменения в рабочую директорию и staging area. Однако она **не создаёт коммит** — это просто изменение файлов, которые затем нужно закоммитить вручную.

### Простое применение патча

```bash
# Применить патч из файла
git apply changes.patch

# Применить патч со сдвигом (если строки немного смещены)
git apply --3way changes.patch

# Применить патч, игнорируя изменения пробелов
git apply --ignore-whitespace changes.patch
```

### Проверка патча перед применением

```bash
# Проверить, применится ли патч без ошибок (не применять)
git apply --check changes.patch

# Если команда выполнилась без ошибок, патч можно безопасно применять
# Если есть ошибки, вы получите сообщение об этом

# Показать статистику изменений без применения
git apply --stat changes.patch

# Результат:
# src/auth.js       | 9 +++++++++
# test/auth.test.js | 6 ++++++
# 2 files changed, 15 insertions(+)
```

### Применение патча с конфликтами

Если патч не применяется полностью, используйте флаг `--3way`:

```bash
# Флаг --3way использует трёхстороннее слияние (наш/общий/их)
# Это даёт больше информации и помогает разрешить конфликты
git apply --3way patch-with-conflicts.patch
```

После применения патча с помощью `git apply` нужно закоммитить изменения:

```bash
git add .
git commit -m "Apply patch: add username validation"
```

## Применение патча: git am (с сохранением коммита и автора)

Команда `git am` (apply mailbox) — это более высокоуровневый инструмент, чем `git apply`. Она применяет патч **и создаёт коммит** со всеми метаданными (автор, дата, сообщение) из формата `git format-patch`.

### Применить один патч

```bash
# Применить патч и создать коммит
git am 0001-feat-add-login.patch

# Результат:
# Applying: feat: add username validation to login function
```

### Применить несколько патчей подряд

```bash
# Применить все патчи из директории по порядку
git am patches/*.patch

# Или применить в конкретном порядке
git am patches/0001-*.patch patches/0002-*.patch patches/0003-*.patch

# Git будет применять их одновременно и создаст коммит для каждого
```

### Что происходит при применении git am

Когда вы выполняете `git am`:

1. Git читает патч в формате электронной почты
2. Извлекает информацию об авторе, дате, сообщении коммита
3. Применяет дифф к файлам
4. Создаёт новый коммит с сохранённой информацией об авторе
5. Оригинальный автор указывается в коммите

Это критически важно в проектах с открытым исходным кодом, где вклады приходят через список рассылки.

## git apply vs git am — таблица сравнения

| Характеристика | git apply | git am |
|---|---|---|
| Создаёт коммит | ❌ Нет | ✅ Да |
| Сохраняет автора | ❌ Нет | ✅ Да |
| Сохраняет дату | ❌ Нет | ✅ Да |
| Сохраняет сообщение | ❌ Нет | ✅ Да |
| Формат входного файла | Любой diff | format-patch (email) |
| Когда использовать | Применить изменения локально | Включить вклад от другого разработчика |
| Команда создания | git diff | git format-patch |

## Типичные ошибки при применении патча

### Ошибка: patch does not apply

```bash
$ git apply changes.patch
error: patch does not apply
```

Это означает, что патч не может быть применён, потому что строки кода не совпадают.

**Решение 1: использовать флаг --3way**

```bash
git apply --3way changes.patch
```

Флаг `--3way` использует информацию о контексте для более умного применения патча.

**Решение 2: применить с игнорированием пробелов**

```bash
git apply --ignore-whitespace changes.patch
git apply --ignore-space-change changes.patch
```

### Ошибка: patch failed при git am

```bash
$ git am 0001-feat-new-feature.patch
error: patch failed: src/main.c:42
error: src/main.c: patch does not apply
Patch failed at 0001 feat: new feature
```

**Решение:**

```bash
# Вы находитесь в состоянии "am in progress"
# Проверьте статус
git status

# Выберите один из вариантов:

# 1. Разрешить конфликт вручную и продолжить
# (отредактируйте файлы в конфликте)
git add .
git am --continue

# 2. Отменить применение патча и вернуться в исходное состояние
git am --abort

# 3. Полностью пропустить этот патч и перейти к следующему
git am --skip
```

### Ошибка: автор патча не совпадает

Если вы получили патч, созданный другим инструментом (не `git format-patch`), в нём может не быть информации об авторе:

```bash
git am --ignore-author-date patched-file.patch
```

## Применение патча с GitHub

GitHub позволяет скачать любой коммит или pull request в виде патча.

### Применить патч из конкретного коммита

```bash
# Замените USER, REPO и COMMIT_SHA
curl https://github.com/USER/REPO/commit/COMMIT_SHA.patch | git am

# Пример:
curl https://github.com/torvalds/linux/commit/abc123def456.patch | git am
```

### Применить патч из pull request

```bash
# Замените USER, REPO и PR_NUMBER
curl https://github.com/USER/REPO/pull/PR_NUMBER.patch | git am

# Пример:
curl https://github.com/nodejs/node/pull/42.patch | git am
```

### Применить с сохранением в файл

```bash
# Сначала скачать в файл
curl https://github.com/USER/REPO/commit/abc123.patch -o patch.patch

# Затем применить
git am patch.patch

# Или с проверкой перед применением
git apply --check patch.patch
git am patch.patch
```

## Часто задаваемые вопросы (FAQ)

**В: Когда нужно использовать патчи вместо pull request?**

О: Патчи полезны, когда вы не имеете доступа к репозиторию, работаете через электронную почту (как в ядре Linux), или когда нужно передать изменения в закрытой системе без интернета. В современной разработке с GitHub патчи используются реже, но остаются важным инструментом в Open Source.

**В: Как отправить изменения коллеге, который не имеет доступа к репозиторию?**

О: Создайте патч и отправьте коллеге файлом:
```bash
git diff > my-changes.patch
# отправить my-changes.patch по email или через файловую систему
# коллега применит:
git apply my-changes.patch
```

**В: Я применил патч, но появились конфликты. Как их исправить?**

О: Если вы использовали `git am` и возникли конфликты, отредактируйте конфликтующие файлы, разрешите конфликты, затем выполните:
```bash
git add .
git am --continue
```

**В: В чём разница между git diff и git format-patch для создания патча?**

О: `git diff` создаёт обычный дифф-файл, который можно применить с помощью `git apply`, но без сохранения метаданных коммита. `git format-patch` создаёт патч в формате электронной почты со всеми метаданными (автор, дата, сообщение), который можно применить с помощью `git am`.

**В: Можно ли применить патч, созданный из одного репозитория, в другой репозиторий?**

О: Да, можно, но это работает лучше всего, если файлы имеют похожую структуру. Используйте флаг `--3way` для более умного применения. Если патч содержит информацию о конкретных путях файлов, которых нет в целевом репозитории, вам придётся редактировать патч вручную.

## Внутренние ссылки

Узнайте больше о связанных командах:
- {{< relref "git-diff" >}} — подробно о команде git diff и сравнении версий
- {{< relref "git-cherry-pick" >}} — применение отдельных коммитов из другой ветки
- {{< relref "git-commit-amend" >}} — редактирование коммитов

## Заключение

Патчи — это мощный инструмент для обмена изменениями в коде, особенно в проектах с открытым исходным кодом. Команда `git apply` подходит для простого применения изменений, а `git am` лучше использовать, когда нужно сохранить информацию об авторе и коммите. Выбирайте инструмент в зависимости от вашего рабочего процесса и требований проекта.
