---
title: "git clean: как удалить неотслеживаемые файлы из рабочей директории"
description: "git clean удаляет untracked файлы из рабочей директории. Флаги -fd, dry run (-n), решение ошибки overwritten."
date: 2025-12-11
lastmod: 2025-12-11
draft: false
slug: "git-clean"
keywords: ["git clean", "git remove untracked files", "git delete untracked files", "git remove unstaged changes", "untracked files git удалить", "git clean fd"]
tags: ["git", "beginner"]
categories: ["git"]
---

Команда `git clean` удаляет неотслеживаемые файлы из вашей рабочей директории. Неотслеживаемые файлы — это файлы, которые Git не знает, потому что они никогда не были добавлены в индекс (staging area) и не закоммичены.

**Важно:** `git clean` необратима. Удалённые файлы не могут быть восстановлены через Git (только через систему восстановления файлов ОС). Всегда используйте dry run (`-n` флаг) перед фактическим удалением!

**Различие между git clean и git restore:**

- `git clean` удаляет неотслеживаемые файлы (которые Git не знает)
- `git restore` откатывает изменения в отслеживаемых файлах (которые уже в репозитории)

## Флаги git clean — шпаргалка

Вот таблица основных флагов `git clean`:

| Флаг | Название | Описание |
|------|----------|---------|
| `-n` | `--dry-run` | Просмотр того, что будет удалено (без реального удаления) |
| `-f` | `--force` | Обязателен для фактического удаления файлов |
| `-d` | | Также удаляет пустые (и содержащие только удаляемые файлы) директории |
| `-x` | | Также удаляет файлы, игнорируемые .gitignore |
| `-X` | | Удаляет только файлы, игнорируемые .gitignore |
| `-i` | `--interactive` | Интерактивный режим — Git спрашивает для каждого файла |

**Почему нужна `-f` (force)?**

По умолчанию Git требует явного подтверждения при удалении файлов, чтобы предотвратить случайную потерю данных. Вы должны явно указать `-f`, чтобы выполнить удаление.

```bash
# Без -f не сработает
git clean
# fatal: clean.requireForce defaults to true and neither -i, -n, or -f was specified; refusing to act.

# Нужно указать -f
git clean -f
```

## Просмотр без удаления (dry run)

**Всегда начинайте с dry run!** Это показывает, что будет удалено, без фактического удаления.

```bash
git clean -n
```

**Пример:**

```bash
$ git clean -n
Would remove build/
Would remove dist/
Would remove node_modules/
Would remove .tmp/
```

**Для удаления файлов и директорий (dry run):**

```bash
git clean -nfd
```

**Пример:**

```bash
$ git clean -nfd
Would remove build/
Would remove dist/
Would remove node_modules/
Would remove .tmp/
Would remove temp.log
Would remove config.local.js
```

Убедитесь, что все перечисленные файлы действительно хотите удалить, прежде чем выполнять команду без `-n`.

## Основные команды удаления

**Удалить только файлы (не директории):**

```bash
git clean -f
```

Это удалит все неотслеживаемые файлы в текущей директории и её подпапках, но оставит пустые директории.

**Пример:**

```bash
$ git status
On branch main
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        README.backup
        config.tmp
        test.log

$ git clean -f
Removing README.backup
Removing config.tmp
Removing test.log

$ git status
On branch main
nothing to commit, working tree clean
```

**Удалить файлы и директории:**

```bash
git clean -fd
```

Это удалит все неотслеживаемые файлы И директории. Используйте это в большинстве случаев.

**Пример:**

```bash
$ git status
On branch main
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        build/
        dist/
        temp/
        config.backup

$ git clean -fd
Removing build/
Removing dist/
Removing temp/
Removing config.backup

$ git status
On branch main
nothing to commit, working tree clean
```

**Полная очистка — удалить ВСЁ, включая gitignore файлы:**

```bash
git clean -fdx
```

Это удалит файлы, указанные в `.gitignore`. Используйте это осторожно, так как это может удалить важные файлы конфигурации.

**Пример:**

```bash
$ cat .gitignore
node_modules/
.env
build/
*.log

$ git status
On branch main
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        .env
        node_modules/
        test.log

$ git clean -fdx
Removing .env
Removing node_modules/
Removing test.log

$ git status
On branch main
nothing to commit, working tree clean
```

**Только удалить файлы из .gitignore:**

```bash
git clean -fdX
```

Это удалит ТОЛЬКО файлы, указанные в `.gitignore`, но оставит обычные неотслеживаемые файлы. Полезно для очистки артефактов сборки при сохранении других файлов.

## Полная очистка как после git clone

Если вы хотите полностью вернуть репозиторий в состояние после свежего клонирования, используйте комбинацию двух команд:

```bash
# Шаг 1: удалить все неотслеживаемые файлы
git clean -fdx

# Шаг 2: откатить все изменения отслеживаемых файлов
git reset --hard HEAD
```

**Различие между этими командами:**

- `git clean -fdx` удаляет неотслеживаемые файлы (файлы, которых никогда не было в репозитории)
- `git reset --hard HEAD` откатывает изменения отслеживаемых файлов на версию из HEAD

**Пример полной очистки:**

```bash
$ git status
On branch main
Your branch is behind 'origin/main' by 3 commits.

Changes not staged for commit:
  (use "git add <file>..." to update what will be included in what will be committed)
        modified:   src/main.js
        modified:   config.json

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        build/
        .env.local
        temp.log

$ git clean -fdx
Removing build/
Removing .env.local
Removing temp.log

$ git reset --hard HEAD
HEAD is now at abc1234 Fix authentication bug

$ git status
On branch main
Your branch is behind 'origin/main' by 3 commits.
  (use "git pull" to update the branch)

nothing to commit, working tree clean
```

**Важно:** `git reset --hard HEAD` откатит только изменения до последнего коммита. Если вам нужно вернуться к состоянию удалённой ветки, используйте:

```bash
git clean -fdx
git reset --hard origin/main
```

## Интерактивный режим (-i)

Для более контролируемого удаления используйте интерактивный режим:

```bash
git clean -fdi
```

Git спросит для каждого файла или группы файлов, что делать.

**Пример сессии:**

```bash
$ git clean -fdi
Would remove the following item(s):
        build/
        dist/
        .env
        temp.log

Remove? [y/N/q/all/] a
Removing build/
Removing dist/
Removing .env
Removing temp.log
Done.
```

**Опции интерактивного режима:**

- `y` (yes) — удалить этот элемент
- `n` (no) — не удалять
- `q` (quit) — выход из режима
- `a` (all) — удалить все оставшиеся элементы
- `d` (don't) — не удалять ничего из оставшихся
- `?` — показать справку

Это медленнее, чем простое `git clean -fd`, но даёт вам полный контроль над процессом.

## Ошибка: untracked working tree files would be overwritten by checkout

Эта ошибка возникает, когда вы пытаетесь переключиться на другую ветку, а в текущей рабочей директории есть неотслеживаемые файлы, которые конфликтуют с файлами в целевой ветке.

**Пример ошибки:**

```bash
$ git checkout feature/new-feature
error: The following untracked working tree files would be overwritten by checkout:
    src/config.js
    utils/helper.js
Please move or remove these files before you switch branches.
```

Это означает, что в ветке `feature/new-feature` есть отслеживаемые файлы `src/config.js` и `utils/helper.js`, а в вашей текущей рабочей директории есть неотслеживаемые файлы с теми же именами. Git не может переключиться на другую ветку, потому что это перезапишет ваши файлы.

**Решение 1: удалить конфликтующие файлы**

```bash
git clean -fd
git checkout feature/new-feature
```

**Решение 2: переместить файлы в другое место**

```bash
mv src/config.js src/config.js.backup
mv utils/helper.js utils/helper.js.backup
git checkout feature/new-feature
```

**Решение 3: добавить файлы в .gitignore и использовать git clean -fdx**

Если эти файлы должны быть проигнорированы:

```bash
echo "src/config.js" >> .gitignore
echo "utils/helper.js" >> .gitignore
git clean -fdx
git checkout feature/new-feature
```

**Решение 4: использовать git stash вместо clean**

Если файлы важны и вы не хотите их удалять, используйте stash:

```bash
git stash push -u  # -u = include untracked files
git checkout feature/new-feature
# позже восстановите
git stash pop
```

## git clean vs git restore vs git reset

Часто возникает путаница между этими тремя командами. Вот таблица различий:

| Команда | Что удаляет | Отслеживаемые файлы | Примечание |
|---------|------------|-----------------|-----------|
| `git clean -fd` | Неотслеживаемые файлы и директории | Не трогает | Удаляет файлы, которых никогда не было в Git |
| `git restore .` | Изменения в отслеживаемых файлах | Откатывает все изменения | Восстанавливает файлы из индекса (staging area) |
| `git restore --staged .` | Изменения в staging area | Убирает файлы из индекса | Не меняет сами файлы, только их статус |
| `git reset --hard HEAD` | Изменения в отслеживаемых файлах | Откатывает в HEAD | Возвращает файлы к последнему коммиту |

**Практические примеры:**

```bash
# Случай 1: вы создали новый файл test.js (неотслеживаемый)
# и хотите его удалить
git clean -f

# Случай 2: вы изменили main.js (отслеживаемый)
# и хотите откатить изменения
git restore main.js

# Случай 3: вы добавили файл в staging area (git add)
# и хотите убрать его оттуда
git restore --staged main.js

# Случай 4: вы сделали несколько коммитов и хотите
# откатиться на 3 коммита назад
git reset --hard HEAD~3
```

## Восстановление после git clean

**Важная информация:** если вы выполнили `git clean` без dry run, файлы удаляются навсегда из Git. Они не будут видны в reflog, потому что они никогда не были закоммичены.

**Единственный способ восстановления:**

Используйте встроенные инструменты восстановления файлов вашей операционной системы:

**Linux/macOS:**

```bash
# Файлы могут быть в папке Trash
ls ~/.Trash/

# Или используйте специальные инструменты восстановления
extundelete /dev/sdX
```

**Windows:**

1. Откройте Recycle Bin (Корзина)
2. Найдите удалённые файлы
3. Кликните правой кнопкой и выберите "Restore"

**macOS:**

1. Откройте Finder
2. Перейдите в Trash
3. Найдите файлы
4. Кликните правой кнопкой и выберите "Put Back"

**Профилактика — всегда использовать dry run:**

Это главная причина, почему так важно использовать `git clean -n` перед фактическим удалением. Один раз проверили, что будет удаляться, — и вы в безопасности:

```bash
# Всегда сначала
git clean -nfd

# Потом, если всё ОК
git clean -fd
```

## Типичные ошибки

**Ошибка 1: "fatal: clean.requireForce defaults to true"**

```bash
$ git clean
fatal: clean.requireForce defaults to true and neither -i, -n, or -f was specified; refusing to act.
```

**Решение:** используйте `-f` флаг для фактического удаления или `-n` для просмотра.

**Ошибка 2: удалены важные файлы конфигурации**

Если вы выполнили `git clean -fdx` и удалили `.env` или другие файлы, которые должны быть проигнорированы:

1. Добавьте их в `.gitignore`
2. Восстановите файлы из резервной копии (если есть)
3. В будущем используйте `git clean -fd` без `-x` флага

**Ошибка 3: git clean удалил всю папку node_modules**

```bash
$ git clean -fd
Removing node_modules/

$ npm install
# пересоздаёт папку
```

Это нормально, если `node_modules/` в `.gitignore`. Используйте `npm install` для её восстановления.

**Ошибка 4: забыли сделать dry run**

Если вы случайно удалили файлы без проверки:

```bash
# Попробуйте восстановить через ОС
# (см. раздел "Восстановление после git clean")

# В будущем используйте alias для безопасности
git config alias.cleancheck 'clean -nfd'
git cleancheck  # это safe, просто просмотр
```

## FAQ: часто задаваемые вопросы

**1. Какая разница между `-f` и `-fd`?**

- `git clean -f` удаляет только неотслеживаемые файлы
- `git clean -fd` удаляет файлы И пустые (или содержащие только удаляемые файлы) директории

Используйте `-fd` в большинстве случаев, потому что обычно вам нужно удалить и файлы, и папки (например, `build/`, `dist/`, `node_modules/`).

**2. Можно ли восстановить файлы после git clean?**

К сожалению, нет — через Git это невозможно. Файлы, удалённые `git clean`, не были закоммичены, поэтому они не сохранены в истории Git. Единственный способ восстановления — использовать инструменты восстановления файлов вашей ОС (Trash, Time Machine и т.д.).

Это главная причина, почему важно всегда делать dry run перед удалением.

**3. Удаляет ли git clean файлы из .gitignore?**

По умолчанию нет. `git clean` удаляет только неотслеживаемые файлы.

Чтобы удалить файлы из `.gitignore`, используйте:
- `git clean -fdx` — удалить все неотслеживаемые И игнорируемые файлы
- `git clean -fdX` — удалить ТОЛЬКО игнорируемые файлы

**4. Как очистить только конкретную директорию?**

Используйте path после команды:

```bash
# Очистить только папку build/
git clean -fd build/

# Очистить всё кроме определённой папки
git clean -fd --exclude=node_modules/
```

**5. Как сделать git clean более безопасной по умолчанию?**

Создайте alias:

```bash
# Alias для безопасного просмотра
git config alias.cleancheck 'clean -nfd'

# Alias для безопасного удаления (требует подтверждения)
git config alias.cleanit 'clean -i'

# Теперь используйте
git cleancheck     # просмотр (безопасно)
git cleanit        # интерактивное удаление (с вопросами)
```

## Резюме

`git clean` — это мощный инструмент для удаления неотслеживаемых файлов:

**Основные команды:**

```bash
# Просмотр без удаления (ВСЕГДА НАЧНИТЕ ОТСЮДА)
git clean -nfd

# Удалить файлы и папки
git clean -fd

# Удалить файлы, папки и файлы из .gitignore
git clean -fdx

# Интерактивное удаление (безопаснее)
git clean -fdi
```

**Запомните:**

1. **Всегда используйте `-n` (dry run) перед удалением**
2. **Флаг `-f` обязателен для фактического удаления**
3. **Удаление необратимо** — восстановление возможно только через ОС
4. **Используйте `-i` для интерактивного режима**, если не уверены

## Внутренние ссылки

Для более полного понимания управления файлами в Git рекомендуем прочитать:

- {{< relref "otmenit-izmeneniya-fajl-git" >}} — как откатывать изменения отслеживаемых файлов
- {{< relref "gitignore" >}} — как использовать .gitignore для игнорирования файлов
- {{< relref "git-stash" >}} — как временно сохранять изменения с помощью stash
