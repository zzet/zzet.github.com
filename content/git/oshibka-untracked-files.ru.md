---
title: "Ошибка «untracked working tree files would be overwritten»"
description: "Как исправить ошибку «untracked working tree files would be overwritten by checkout/merge» в Git. Причины ошибки и три способа решения."
date: 2024-12-02
lastmod: 2024-12-02
draft: false
slug: "oshibka-untracked-files"
keywords: ["error the following untracked working tree files would be overwritten by checkout", "untracked working tree files would be overwritten by merge", "git the following untracked working tree files would be overwritten", "commit or discard the untracked or modified content in submodules"]
tags: ["git", "errors", "intermediate"]
categories: ["git"]
aliases: []
---

Эта ошибка пугает многих разработчиков, но на самом деле она полезна — Git защищает вас от потери данных. В этой статье разберёмся, что означает эта ошибка, почему она возникает и как её исправить.

## Что означает ошибка "untracked working tree files would be overwritten"

Когда вы видите сообщение об ошибке, оно выглядит примерно так:

```
error: The following untracked working tree files would be overwritten by checkout:
        config/database.yml
        node_modules/
Please move or remove these files before you switch branches.
```

Или при выполнении merge/pull:

```
error: The following untracked working tree files would be overwritten by merge:
        .env
        tmp/cache
Please move or remove these files before you merge.
Aborting
```

### Почему это происходит

Git не может переключиться на другую ветку (или выполнить merge/pull), потому что:

1. **В текущей рабочей директории есть файлы, которые Git не отслеживает** (untracked files)
2. **В целевой ветке эти же файлы уже существуют** и отслеживаются Git
3. **Git не хочет перезаписывать ваши файлы** и потенциально потерять данные

Проще говоря: вы создали локальный файл `config/database.yml`, но в другой ветке уже есть отслеживаемый файл с тем же именем. Git не знает, нужно ли ему перезаписать вашу версию.

## Вариант 1: Ошибка при git checkout / git switch

Вы пытаетесь переключиться на другую ветку:

```bash
git checkout main
# или
git switch main
```

И получаете ошибку выше.

### Решение 1: Закоммитить файлы

Если файлы нужны вам в проекте, добавьте их в git и закоммитьте:

```bash
# Добавить все файлы
git add .

# Закоммитить с понятным сообщением
git commit -m "Add config and cache files"

# Теперь можно переключиться на другую ветку
git checkout main
```

Это безопасный вариант, потому что ваши изменения сохранены в истории Git.

### Решение 2: Переместить в stash (временное хранилище)

Если вы ещё не уверены, нужны ли эти файлы, используйте `git stash`:

```bash
# Переместить untracked файлы в stash
git stash push config/database.yml node_modules/

# Или все untracked файлы сразу
git stash push -u

# Теперь переключаемся
git checkout main

# Позже вернуть файлы обратно
git stash pop
```

Подробнее о stash можно прочитать в статье {{< relref "git-stash" >}}.

### Решение 3: Удалить файлы (если они не нужны)

Если эти файлы временные (логи, кэши, IDE-конфиги), просто удалите их:

```bash
# Удалить конкретные файлы
rm config/database.yml
rm -rf node_modules/

# Или удалить все untracked файлы (будьте осторожны!)
git clean -fd
```

**Предупреждение:** `git clean -fd` удаляет все untracked файлы безвозвратно! Сначала используйте `git clean -n` для предпросмотра.

## Вариант 2: Ошибка при git pull / git merge

Эта ошибка также возникает при попытке слить ветки:

```bash
git pull origin main
# error: The following untracked working tree files would be overwritten by merge...
```

Или при явном merge:

```bash
git merge develop
# error: The following untracked working tree files would be overwritten by merge...
```

### Три способа решения

**Способ 1: Закоммитить изменения**

```bash
git add .
git commit -m "Work in progress"
git pull origin main
```

**Способ 2: Переместить в stash**

```bash
git stash push -u
git pull origin main
git stash pop
```

**Способ 3: Удалить временные файлы**

```bash
# Сначала посмотреть что будет удалено
git clean -n

# Если всё нормально, удалить
git clean -fd

# Теперь merge пройдёт успешно
git pull origin main
```

### Специальный случай: git clean

Команда `git clean` удаляет все untracked файлы. Существует несколько вариантов:

```bash
# Только просмотр (-n = --dry-run)
git clean -n

# Удалить файлы (но не директории)
git clean -f

# Удалить файлы и пустые директории
git clean -fd

# Удалить файлы, директории и ignored файлы
git clean -fdx

# Интерактивный режим (спросит перед удалением)
git clean -i
```

## Вариант 3: Ошибка в submodule

Если вы используете {{< relref "git-submodules" >}}, ошибка может выглядеть так:

```
error: The following untracked working tree files would be overwritten by checkout:
        submodule-name/file.txt
Please move or remove these files before you switch branches.
Aborting
Stopping at 'submodule-name'; stopping the work tree for this directory...
failed to traverse into the directory 'submodule-name'
Failed to merge submodule submodule-name
```

Или:

```
commit or discard the untracked or modified content in submodules
```

### Решение для submodule

**Шаг 1:** Войти в директорию субмодуля

```bash
cd submodule-name
```

**Шаг 2:** Посмотреть статус

```bash
git status
```

**Шаг 3:** Либо закоммитить, либо откатить изменения

```bash
# Вариант 1: Закоммитить в субмодуле
git add .
git commit -m "Update submodule"

# Вариант 2: Откатить все изменения в субмодуле
git checkout .

# Вариант 3: Удалить untracked файлы
git clean -fd
```

**Шаг 4:** Вернуться в главный проект и обновить субмодули

```bash
cd ..
git submodule update --init --recursive
git pull origin main
```

## Профилактика: правильно настроить .gitignore

Лучше всего предотвратить эту ошибку, правильно настроив файл `.gitignore`. Таким образом, Git не будет отслеживать временные файлы с самого начала.

### Примеры правил .gitignore

Создайте файл `.gitignore` в корне проекта:

```gitignore
# IDE
.vscode/
.idea/
*.swp
*.swo

# Node.js
node_modules/
npm-debug.log
yarn-error.log

# Python
__pycache__/
*.py[cod]
venv/
.env

# Конфиги (особенно с приватной информацией)
config/database.yml
config/secrets.yml
.env
.env.local

# Build директории
build/
dist/
*.o
*.a

# OS
.DS_Store
Thumbs.db

# Логи
*.log
logs/

# Временные файлы
tmp/
temp/
*.tmp
```

Подробнее о `.gitignore` читайте в статье {{< relref "gitignore" >}}.

### Совет: используйте глобальный .gitignore

Вы также можете создать глобальный файл `.gitignore` для всех проектов на вашем компьютере:

```bash
# Создать глобальный gitignore
git config --global core.excludesfile ~/.gitignore_global

# Добавить правила для вашей IDE
echo ".vscode/" >> ~/.gitignore_global
echo ".idea/" >> ~/.gitignore_global
echo ".DS_Store" >> ~/.gitignore_global
```

## Пошаговая инструкция по решению ошибки

Если вы столкнулись с этой ошибкой прямо сейчас, вот быстрая схема решения:

1. **Посмотрите статус:**
   ```bash
   git status
   ```

2. **Выберите способ решения:**
   - Если файлы нужны: `git add . && git commit -m "..."`
   - Если файлы временные: `git clean -fd`
   - Если не уверены: `git stash push -u`

3. **Повторите команду:**
   ```bash
   git checkout main  # или git pull, или git merge
   ```

## Частые вопросы (FAQ)

### Что означает "untracked working tree files would be overwritten by checkout"?

Это значит, что у вас есть файлы, которые Git не отслеживает (вы их не добавили в `git add`), но в ветке, на которую вы пытаетесь переключиться, эти файлы уже существуют. Git не хочет их перезаписывать.

### Как исправить ошибку "untracked files would be overwritten by merge"?

Три способа:
1. Закоммитить: `git add . && git commit -m "..."`
2. В stash: `git stash push -u && git pull`
3. Удалить: `git clean -fd && git pull`

### Безопасен ли git clean -fd?

**Нет!** `git clean -fd` безвозвратно удаляет все untracked файлы. Всегда сначала используйте `git clean -n` для предпросмотра, чтобы убедиться, что вы удаляете нужные файлы.

### Как переключиться на ветку с несохранёнными изменениями?

Если у вас есть tracked файлы с изменениями, используйте stash:

```bash
git stash
git checkout main
git stash pop  # или git stash apply
```

Если это untracked файлы, они не будут сохранены в stash по умолчанию. Используйте флаг `-u`:

```bash
git stash push -u
```

### Почему Git не даёт переключить ветку?

Git защищает вас от потери данных. Если у вас есть untracked файлы, которые конфликтуют с целевой веткой, Git не будет переключаться, пока вы не избавитесь от конфликта.

### Как предотвратить ошибку untracked files?

Используйте правильный `.gitignore` файл:
- Добавьте правила для `node_modules/`, `build/`, `.env`, IDE-конфигов
- Используйте глобальный `.gitignore` для персональных правил
- Регулярно проверяйте статус: `git status`

### Можно ли восстановить удалённые файлы после git clean?

К сожалению, **нет**. `git clean` полностью удаляет файлы из файловой системы, и они не будут в Git-истории (потому что они были untracked). Будьте осторожны с этой командой!

### В чём разница между git stash и git clean?

- **git stash:** сохраняет изменения во временное хранилище, файлы остаются в файловой системе
- **git clean:** полностью удаляет файлы из файловой системы

Используйте `git stash`, если вы не уверены в необходимости удаления.

## Заключение

Ошибка "untracked working tree files would be overwritten" — это защитный механизм Git, который предотвращает потерю данных. Если вы её видите:

1. **Проверьте статус:** `git status`
2. **Выберите способ решения:** закоммитить, stash или удалить
3. **Предотвращайте в будущем:** используйте `.gitignore`

Помните, что Git всегда на вашей стороне — он просто пытается защитить вас от ошибок. Если у вас остались вопросы, используйте `git help` или обратитесь к документации о {{< relref "git-stash" >}} и {{< relref "git-submodules" >}}.
