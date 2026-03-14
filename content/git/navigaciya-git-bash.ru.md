---
title: "Навигация в Git Bash: перемещение между папками"
description: "Простое руководство по навигации в Git Bash. Команда cd, абсолютные и относительные пути, автодополнение, практические примеры."
date: 2026-02-17
lastmod: 2026-02-17
draft: false
slug: "navigaciya-git-bash"
keywords: ["как в git bash перейти в нужную папку", "cd команда git bash", "навигация по папкам", "git bash команды", "перейти в папку git bash", "git bash cd команда"]
tags: ["git", "git-bash", "windows", "beginner"]
categories: ["git"]
---

Первое, с чем сталкиваются новички в Git Bash — навигация по файловой системе. Без умения перемещаться между папками невозможно ни клонировать репозиторий, ни запускать команды в нужном проекте. Этот навык требует только одной основной команды.

## Основная команда: cd (change directory)

`cd` — команда для перемещения между папками. Работает одинаково в Git Bash, macOS Terminal и Linux.

```bash
# Перейти в папку в текущей директории
cd my-project

# Перейти в подпапку
cd projects/my-project

# Перейти в папку с пробелом в имени
cd "My Project"
```

После `cd` подсказка изменится, показывая новый путь.

## Узнать где вы находитесь: pwd

```bash
# Показать текущую папку (Print Working Directory)
pwd

# Вывод в Git Bash на Windows:
# /c/Users/YourName/Documents/my-project
# (Диски C:, D: отображаются как /c, /d)
```

## Просмотр содержимого папки: ls

```bash
# Содержимое текущей папки
ls

# С подробностями (права, размер, дата)
ls -la

# Скрытые файлы (начинающиеся с .)
ls -a

# Содержимое конкретной папки
ls ~/Projects
```

## Абсолютные пути

Абсолютный путь — полный путь от корня файловой системы. На Windows диски обозначаются `/c/` вместо `C:\`:

```bash
# Перейти на диск C в папку Documents
cd C:/Users/YourName/Documents

# Git Bash понимает оба формата:
cd C:/Users/YourName/Documents
cd C:\Users\YourName\Documents

# Через Unix-нотацию
cd /c/Users/YourName/Documents

# Перейти на диск D
cd D:/Projects
cd /d/Projects
```

## Относительные пути

Относительный путь — от текущей папки:

```bash
# Перейти в подпапку
cd src
cd src/components

# Перейти на уровень выше
cd ..

# Два уровня вверх
cd ../..

# Вверх и в соседнюю папку
cd ../other-project

# Перейти в папку рядом с текущей
cd ../sibling-folder/subfolder
```

## Специальные пути

```bash
# Домашняя папка (/c/Users/YourName)
cd ~
cd  # без аргументов — тоже домашняя папка

# Вернуться в предыдущую папку
cd -
# Полезно для переключения между двумя папками

# Корень файловой системы
cd /
```

## Автодополнение (Tab)

Tab — главный инструмент для быстрой навигации. Начните вводить имя папки и нажмите Tab:

```bash
# Начните вводить
cd Docu
# Нажмите Tab
cd Documents  # автодополнение

# Несколько вариантов — нажмите Tab ещё раз
cd Do  # Tab Tab
# Показывает: Documents/ Downloads/
```

Автодополнение работает для папок, файлов и Git команд.

## Практический пример: открыть Git проект

```bash
# 1. Открыть Git Bash (Пуск → Git Bash)
# Вы находитесь в домашней папке

# 2. Посмотреть что здесь есть
ls

# 3. Перейти в папку с проектами
cd Documents/Projects
# или
cd ~/Documents/Projects

# 4. Посмотреть список проектов
ls

# 5. Перейти в нужный проект
cd my-website

# 6. Убедиться что это Git репозиторий
git status
```

## Создание папок

```bash
# Создать папку
mkdir new-project

# Создать папку и сразу перейти в неё
mkdir my-app && cd my-app

# Создать вложенные папки сразу
mkdir -p projects/2024/my-app
```

## Работа с путями содержащими пробелы

Если имя папки или файла содержит пробелы — нужны кавычки:

```bash
# Неправильно (Git Bash поймёт как две отдельные команды)
cd My Projects

# Правильно — в кавычках
cd "My Projects"
cd "My Documents/My Project"

# Или экранировать пробелы обратным слэшем
cd My\ Projects

# Абсолютный путь с пробелами
cd "/c/Users/John Smith/Desktop"
```

## Создание и удаление директорий

```bash
# Создать одну папку
mkdir new-folder

# Создать вложенные папки
mkdir -p parent/child/grandchild
# Флаг -p создаёт все промежуточные папки

# Удалить пустую папку
rmdir empty-folder

# Удалить папку со всем содержимым (осторожно!)
rm -rf my-folder

# Проверить перед удалением
ls -la my-folder  # посмотреть что в папке
rm -rf my-folder  # потом удалить

# Удалить только файлы в папке, саму папку оставить
rm -rf my-folder/*
```

## Операции с файлами

```bash
# Скопировать файл
cp source.txt destination.txt

# Скопировать со всеми вложениями
cp -r source-folder/ destination-folder/

# Переместить/переименовать файл
mv old-name.txt new-name.txt

# Посмотреть содержимое файла
cat file.txt

# Показать содержимое с номерами строк
cat -n file.txt

# Просмотр больших файлов постранично
less file.txt  # Нажимайте Space, q чтобы выход

# Удалить файл
rm file.txt

# Удалить несколько файлов
rm file1.txt file2.txt file3.txt

# Найти файлы по имени
find . -name "*.js"
```

## История команд

```bash
# Просмотреть историю последних команд
history

# Показать последние 20 команд
history 20

# Стрелка вверх/вниз — переключение между командами
# Ctrl+R — поиск в истории (Interactive search)
# Ctrl+C — прервать текущую команду
# Ctrl+A — перейти в начало строки
# Ctrl+E — перейти в конец строки
# Ctrl+L — очистить экран

# После Ctrl+R начните вводить часть команды
# Ctrl+R
# (reverse-i-search)`git': git status
# Нажимайте Ctrl+R для поиска далее или Enter для выполнения
```

## Полезные клавиатурные сокращения

```bash
# Tab — автодополнение имён файлов и папок
cd Doc[Tab] → cd Documents

# Ctrl+A — начало строки
# Ctrl+E — конец строки
# Ctrl+K — удалить всё после курсора
# Ctrl+U — удалить всё перед курсором
# Alt+Backspace — удалить слово
# Ctrl+W — удалить слово перед курсором

# Ctrl+D — выход из Git Bash (или type exit)
# Ctrl+Z — отправить процесс в фон (редко используется)
```

## Переменные окружения и PATH

```bash
# Показать значение переменной
echo $PATH
echo $HOME
echo $USER

# Установить временную переменную
export MY_VAR="my value"
echo $MY_VAR

# PATH — список папок где Git Bash ищет исполняемые файлы
# Если команда не найдена — нужно добавить путь в PATH

# Добавить папку в PATH (временно, только для текущей сессии)
export PATH="$PATH:/c/Program Files/MyApp"

# Проверить где находится команда
which git
# /usr/bin/git

which node
# /c/Program Files/nodejs/node.exe
```

## Настройка Git Bash prompt (.bashrc)

Файл `~/.bashrc` выполняется при старте Git Bash. Используйте его для настройки приглашения команд:

```bash
# Добавить в ~/.bashrc (откройте Notepad++ или код)
# Путь: C:\Users\YourName\.bashrc

# Простое приглашение с веткой Git
parse_git_branch() {
  git branch 2>/dev/null | grep "^*" | sed 's/^* //'
}

PS1='\u@\h \w (\$(parse_git_branch)) $ '
# Результат: user@hostname ~/project (main) $

# Цветной prompt
PS1='\[\033[32m\]\u@\h\[\033[00m\] \[\033[34m\]\w\[\033[00m\] $ '
# Зелёный текст для имени, синий для пути
```

Сохраните файл и перезапустите Git Bash.

## Часто задаваемые вопросы

**Как перейти на диск D: в Git Bash?** `cd D:/` или `cd /d/`. Пример: `cd D:/Projects/my-app`. Git Bash понимает оба формата: с прямыми слэшами и обратными.

**Что значит ../ и когда его использовать?** `..` указывает на родительскую (вышестоящую) папку. `../` — путь к ней. `../../` — два уровня вверх. Используйте для перемещения к соседним проектам: `cd ../../other-project` или возвращения из подпапки.

**Почему Git Bash использует слэши вперёд (/), а не назад (\)?** Это Unix-нотация. Git Bash работает на основе Unix-инструментов и использует Unix-пути. Windows пути автоматически транслируются. Оба формата поддерживаются в Git Bash.

**Как быстро вернуться в предыдущую папку?** `cd -` переключает между двумя последними папками. Очень удобно для переключения между двумя проектами.

**Что делать если пути содержат кириллицу?** Заключить путь в кавычки: `cd "Мои документы"`. Хотя работает, рекомендуется избегать кириллицы в путях к проектам — иногда вызывает проблемы с Git и скриптами.

**Как работает автодополнение (Tab) в Git Bash?** Нажмите Tab когда вводите имя папки или файла. Git Bash автоматически дополнит имя если есть однозначный вариант. Если вариантов несколько — нажмите Tab ещё раз для списка.

**Как очистить экран Git Bash?** Используйте `clear` или `Ctrl+L`. Это не удаляет историю команд, просто прокручивает экран.

**Где находится домашняя папка (~)?** На Windows это `C:\Users\YourName` или `/c/Users/YourName` в Git Bash нотации. В Linux/macOS — `/home/user` или `/Users/user`.

## Поиск файлов и папок в Git Bash

```bash
# Найти все файлы с расширением .js
find . -name "*.js"

# Найти папки с именем node_modules
find . -type d -name "node_modules"

# Найти большие файлы (более 100MB)
find . -size +100M

# Найти файлы изменённые в последний день
find . -mtime -1

# Поиск по содержимому (grep)
grep -r "function name" .
grep -r "TODO" src/

# Поиск файлов с рекурсией по типу
find . -name "*.log" -type f
```

## Сравнение размеров и подробности файлов

```bash
# Размер папки или файла
du -sh my-folder
du -sh file.txt

# Подробная информация о файлах
ls -lah

# Расшифровка:
# -l: detailed list format
# -a: show hidden files (starting with .)
# -h: human-readable sizes (KB, MB, GB)
# -S: sort by size

# Общий размер всех файлов в папке
du -sh .

# Размер каждой папки в текущей директории
du -sh */

# Показать места где много данных
du -h --max-depth=1 | sort -h
```

## Сочетание команд (pipes) в Git Bash

Pipes (`|`) передают вывод одной команды в другую:

```bash
# Показать файлы и подсчитать количество
ls -la | wc -l

# Найти все JS файлы и посчитать
find . -name "*.js" | wc -l

# Показать большие файлы (отсортировано)
ls -lhS | head -10  # top 10 files

# Найти в истории команду содержащую 'git'
history | grep git

# Использовать grep для поиска в выводе
ls -la | grep node_modules
```

## Работа с переменными и подстановки команд

```bash
# Сохранить результат команды в переменную
current_dir=$(pwd)
echo "I am in: $current_dir"

# Подставить результат команды в другую команду
git init $(date +%Y-%m-%d-repo)  # создаст папку типа 2026-03-15-repo

# Количество файлов
num_files=$(find . -type f | wc -l)
echo "Found $num_files files"

# Использовать переменные в условиях
project_name="my-app"
mkdir "$project_name"
cd "$project_name"
git init
```

## Быстрая навигация с пользовательскими alias

Добавить в ~/.bashrc:

```bash
# Полезные alias для частых команд
alias l='ls -lah'
alias p='pwd'
alias back='cd -'
alias projects='cd ~/Projects'
alias work='cd ~/Work'

# Alias для Git команд
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gp='git push'

# Alias для навигации в подпапки
alias src='cd src'
alias dist='cd dist'
alias test='cd __tests__'

# После добавления в ~/.bashrc, перезагрузить:
source ~/.bashrc
```

## Частые проблемы и решения

**Проблема: "command not found"**
```bash
# Команда не найдена (обычно файл не в PATH)
node: command not found

# Решение 1: полный путь к команде
/c/Program\ Files/nodejs/node.exe --version

# Решение 2: добавить в PATH (в ~/.bashrc)
export PATH="$PATH:/c/Program Files/nodejs"
source ~/.bashrc
```

**Проблема: "Permission denied" при выполнении скрипта**
```bash
# Файл не имеет прав на выполнение
./my-script.sh
# bash: ./my-script.sh: Permission denied

# Решение: добавить права на выполнение
chmod +x my-script.sh
./my-script.sh  # теперь работает
```

**Проблема: вывод команды очень большой и быстро листается**
```bash
# Выводит много данных одновременно
ls -la /c/Windows/System32

# Решение: постранично (нажимайте Space, q для выхода)
ls -la /c/Windows/System32 | less

# Или сохранить в файл
ls -la /c/Windows/System32 > output.txt
```

## Заключение

Три команды для начала работы: `pwd` (где я?), `ls` (что здесь?), `cd` (перейти куда-то). Привычка использовать Tab автодополнения делает навигацию быстрой. Пользовательские alias экономят время при частых операциях. Для работы с репозиториями в Git Bash — [подключение к репозиторию через Git Bash]({{< relref "podklyuchitsya-k-repozitoriyu-git-bash" >}}).
