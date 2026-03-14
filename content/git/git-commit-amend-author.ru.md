---
title: "Как изменить автора коммита в Git: имя, email и дата"
description: "Как изменить автора и email коммита в Git: --amend для последнего, rebase -i для нескольких коммитов."
date: 2026-02-09
lastmod: 2026-02-09
draft: false
slug: "git-commit-amend-author"
keywords: ["git change commit author", "изменить автора коммита git", "git amend author", "git commit author change", "git change email commit"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Иногда случается, что вы совершили коммит с неправильным автором. Распространенные причины:

- Вы коммитили на личном лаптопе с рабочей конфигурацией Git
- Перешли на работу и забыли переключить конфигурацию
- Допустили опечатку в имени или email
- Корпоративные требования: все коммиты должны быть от компании
- Работаете на нескольких проектах с разными email адресами

Хорошая новость: Git позволяет изменить автора коммита на любом этапе. Процесс зависит от того, какие коммиты нужно исправить: только последний или несколько.

## Исправить автора последнего коммита

Это самый простой случай. Используйте флаг `--amend`:

### Способ 1: Указать нового автора напрямую

```bash
git commit --amend --author="John Doe <john@example.com>"
```

Это изменяет автора последнего коммита и открывает редактор для редактирования сообщения. Если вам не нужно редактировать сообщение, используйте `--no-edit`:

```bash
git commit --amend --author="John Doe <john@example.com>" --no-edit
```

### Способ 2: Изменить конфигурацию, потом перезаписать

Этот способ удобнее, если вы хотите использовать новый author для будущих коммитов:

```bash
# 1. Обновите конфигурацию Git
git config user.name "John Doe"
git config user.email "john@example.com"

# 2. Переименуйте автора в последнем коммите, используя переменные окружения
# (удобнее чем вводить вручную)
git commit --amend --reset-author --no-edit

# 3. Для будущих коммитов будет использоваться новый author
```

Флаг `--reset-author` особенно полезен, потому что берет информацию о пользователе из текущей конфигурации.

### Способ 3: Изменить только email

Если нужно изменить только email сохраняя имя:

```bash
# 1. Скопируйте текущее имя
git log -1 --format=%an  # получить имя последнего коммита

# 2. Используйте его с новым email
git commit --amend --author="John Doe <newemail@example.com>" --no-edit
```

### Пример

```bash
# Проверим текущий автор
git log -1 --oneline --format="%an <%ae>"
# OldName <old@example.com>

# Исправляем
git commit --amend --author="John Doe <john@example.com>" --no-edit

# Проверяем результат
git log -1 --oneline --format="%an <%ae>"
# John Doe <john@example.com>

# Push (требуется force push так как хеш изменился)
git push origin main --force
```

## Исправить автора нескольких коммитов

Если нужно изменить автора в нескольких коммитах, используйте интерактивный rebase:

### Способ 1: Interactive Rebase с амендом

```bash
# Изменить автора в последних 5 коммитах
git rebase -i HEAD~5
```

Откроется редактор со списком коммитов:

```
pick abc1234 First commit
pick def5678 Second commit
pick ghi9012 Third commit
pick jkl3456 Fourth commit
pick mno7890 Fifth commit
```

Измените `pick` на `edit` для коммитов, в которых нужно изменить автора:

```
pick abc1234 First commit
edit def5678 Second commit
edit ghi9012 Third commit
pick jkl3456 Fourth commit
pick mno7890 Fifth commit
```

Сохраните (`:wq` в vim или Ctrl+S в других редакторах).

Git остановится на каждом `edit` коммите. Для каждого:

```bash
# Когда Git останавливается на коммите, выполните:
git commit --amend --author="John Doe <john@example.com>" --no-edit

# Потом продолжите rebase
git rebase --continue
```

Цикл повторяется для каждого `edit` коммита.

### Способ 2: Скрипт для автоматизации

Если нужно изменить автора в большом количестве коммитов, напишите скрипт:

```bash
#!/bin/bash
# change-author.sh

OLD_NAME="OldName"
OLD_EMAIL="old@example.com"
NEW_NAME="John Doe"
NEW_EMAIL="john@example.com"

git filter-repo \
  --commit-callback '
    if commit.author.name == b"'"$OLD_NAME"'" and commit.author.email == b"'"$OLD_EMAIL"'":
      commit.author.name = b"'"$NEW_NAME"'"
      commit.author.email = b"'"$NEW_EMAIL"'"
  ' \
  --force
```

Запустите:
```bash
bash change-author.sh
```

### Практический пример

```bash
# Проверим историю
git log --oneline --format="%an <%ae>" | head -10
# OldName <old@example.com>
# OldName <old@example.com>
# OldName <old@example.com>
# John Doe <john@example.com>
# ...

# Нужно изменить первые 3 коммита
git rebase -i HEAD~10

# В редакторе меняем первые 3 на "edit"
# edit abc1234
# edit def5678
# edit ghi9012
# pick ...

# Git останавливается на первом edit коммите
git commit --amend --author="John Doe <john@example.com>" --no-edit
git rebase --continue

# То же самое для второго коммита
git commit --amend --author="John Doe <john@example.com>" --no-edit
git rebase --continue

# И для третьего
git commit --amend --author="John Doe <john@example.com>" --no-edit
git rebase --continue

# Готово! Проверяем результат
git log --oneline --format="%an <%ae>" | head -10
# John Doe <john@example.com>
# John Doe <john@example.com>
# John Doe <john@example.com>
# John Doe <john@example.com>
# ...
```

## Изменить автора во ВСЕЙ истории глобально

Если нужно изменить автора во всех коммитах репозитория (например, при миграции с одного компьютера на другой), используйте `git-filter-repo`.

### Установка git-filter-repo

```bash
# Требуется Python 3

# На macOS
brew install git-filter-repo

# На Linux
pip3 install git-filter-repo

# На Windows
pip install git-filter-repo
```

### Изменение автора во всей истории

```bash
# Установите новую конфигурацию
git config user.name "John Doe"
git config user.email "john@example.com"

# Измените автора для всех коммитов
git filter-repo --name-callback 'return b"John Doe"' \
                 --email-callback 'return b"john@example.com"' \
                 --force

# Проверяем результат
git log --oneline --format="%an <%ae>" | head -5
# John Doe <john@example.com>
# John Doe <john@example.com>
# ...
```

### Более сложный пример: изменить автора только для конкретного email

```bash
# Если ваша старая конфигурация была неправильной
git filter-repo --commit-callback '
    import re
    if commit.author.email == b"old@example.com":
        commit.author.name = b"John Doe"
        commit.author.email = b"john@example.com"
        commit.committer.name = b"John Doe"
        commit.committer.email = b"john@example.com"
' --force
```

**Важно:** После использования `git-filter-repo` нужно сделать force push:

```bash
# Backup на случай ошибки
git branch backup

# Force push (очень осторожно!)
git push origin --force --all
git push origin --force --tags
```

## Настройка разных авторов для разных репозиториев

Лучший способ избежать этой проблемы в будущем — настроить разные авторы для разных проектов.

### Вариант 1: Локальная конфигурация для каждого репозитория

```bash
# Для проекта работы
cd ~/work/company-project
git config user.name "John Doe"
git config user.email "john@company.com"

# Для личного проекта
cd ~/personal/hobby-project
git config user.name "John Doe"
git config user.email "personal@gmail.com"

# Проверим (без --global)
git config user.email
# personal@gmail.com (если вы в hobby-project)
```

### Вариант 2: Глобальная конфигурация с переопределением

```bash
# Глобальная конфигурация (по умолчанию)
git config --global user.name "John Doe"
git config --global user.email "personal@gmail.com"

# Для конкретного репозитория переопределить
cd ~/work/company-project
git config user.name "John Doe"
git config user.email "john@company.com"
```

### Вариант 3: Условное включение конфигурации (includeIf)

Самый удобный способ — использовать `includeIf` в `.gitconfig`:

```bash
# Отредактируйте ~/.gitconfig
nano ~/.gitconfig
```

Добавьте:

```ini
[user]
    name = John Doe
    email = personal@gmail.com

# Переопределить для всех проектов в ~/work/
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

# Переопределить для всех проектов в ~/freelance/
[includeIf "gitdir:~/freelance/"]
    path = ~/.gitconfig-freelance
```

Создайте файлы для разных конфигураций:

```bash
# ~/.gitconfig-work
cat > ~/.gitconfig-work << EOF
[user]
    name = John Doe
    email = john@company.com
EOF

# ~/.gitconfig-freelance
cat > ~/.gitconfig-freelance << EOF
[user]
    name = John Doe
    email = freelance@client.com
EOF
```

Проверьте что работает:

```bash
# В проекте из ~/work
cd ~/work/company-project
git config user.email
# john@company.com

# В личном проекте
cd ~/hobby-project
git config user.email
# personal@gmail.com
```

Это решение работает идеально и вам больше не нужно помнить переключать конфигурацию!

## Различие между author и committer

В Git есть два разных поля в коммите:

```
author: тот кто написал код
committer: тот кто сделал коммит
```

Обычно они одинаковые. Но когда вы используете `git commit --amend`, может измениться только author. Если нужно изменить committer:

```bash
# Измените оба
git commit --amend --author="John Doe <john@example.com>" \
                     --date="2026-02-09 10:00:00" \
                     --no-edit

# Или используйте переменные окружения
GIT_AUTHOR_NAME="John Doe" \
GIT_AUTHOR_EMAIL="john@example.com" \
GIT_COMMITTER_NAME="John Doe" \
GIT_COMMITTER_EMAIL="john@example.com" \
git commit --amend --no-edit
```

## Проверка автора в истории

```bash
# Простой список авторов
git log --oneline --format="%an <%ae>"

# Группировка по авторам (сколько коммитов от каждого)
git shortlog -s -n
# 42  John Doe
# 15  Jane Smith
# 8   Bob Johnson

# Детальный лог с автором
git log --format="%h | %an | %ae | %s" --all

# Для конкретного файла
git log --oneline --format="%an" -- src/app.js | sort | uniq -c
```

## FAQ

**В: Изменится ли хеш коммита после изменения автора?**
О: Да! Хеш коммита зависит от содержимого, автора, даты и других полей. Изменение автора меняет хеш. Это требует force push на сервер.

**В: Безопасно ли делать force push после изменения автора?**
О: Только если вы уверены что никто не скачивал старую историю. В личных branch безопасно. В main/master - обсудите с командой. Обычно лучше создать новый коммит с нужным автором, чем переписывать историю.

**В: Как откатить изменение автора если я ошибся?**
О: Используйте reflog:
```bash
git reflog  # найти старое состояние
git reset --hard <старый-хеш>
```

**В: Могу ли я изменить дату коммита?**
О: Да, вместе с автором:
```bash
git commit --amend --author="John Doe <john@example.com>" \
                     --date="2026-02-09"
```

**В: Что делать если я внес ошибку в filter-repo?**
О: У вас есть backup ветка:
```bash
git reflog  # найти свежую историю
git reset --hard <свежий-хеш>
```

**В: Как изменить только committer, но не author?**
О: Используйте переменные окружения:
```bash
GIT_COMMITTER_NAME="New Name" \
GIT_COMMITTER_EMAIL="new@example.com" \
git commit --amend --no-edit
```

## Наилучшие практики

1. **Предотвращение проблемы:**
   ```bash
   # Сразу при установке Git
   git config --global user.name "John Doe"
   git config --global user.email "john@example.com"
   ```

2. **Для работы и личных проектов:**
   ```bash
   # Используйте includeIf в .gitconfig
   # Автоматически переключает email по директории
   ```

3. **Проверьте перед push:**
   ```bash
   # Перед первым push проверьте автора
   git log --format="%an <%ae>"
   # Убедитесь что это правильно
   ```

4. **Для команды:**
   - Добавьте проверку в pre-commit hook
   - Требуйте определенный формат author
   - Документируйте соглашение (company email или личный?)

## Заключение

Изменение автора коммита в Git просто, но выбор метода зависит от ситуации:

- **Последний коммит:** `git commit --amend --author="..." --no-edit`
- **Несколько коммитов:** `git rebase -i` с `--amend`
- **Вся история:** `git-filter-repo`
- **На будущее:** используйте `includeIf` в .gitconfig

Самое главное — избежать этой проблемы с самого начала, правильно настроив Git конфигурацию для каждого контекста (работа, личные проекты, клиенты).

Читайте также:
- {{< relref "nastrojka-polzovatelya-git" >}} — подробная настройка Git
- {{< relref "git-commit-amend" >}} — как использовать --amend
- {{< relref "git-rebase" >}} — интерактивный rebase и его возможности
