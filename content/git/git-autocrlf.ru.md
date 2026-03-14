---
title: "Настройка git autocrlf: проблемы с переносами строк"
description: "Настройка core.autocrlf в Git для правильной работы с окончаниями строк на Windows и Unix. Как исправить проблемы с CRLF и LF."
date: 2026-01-05
lastmod: 2026-01-05
draft: false
slug: "git-autocrlf"
keywords: ["git crlf", "autocrlf", "core autocrlf", "git autocrlf true false", "git config core autocrlf", "переносы строк git"]
tags: ["git", "windows", "config"]
categories: ["git"]
---

Если вы работаете на Windows в команде с разработчиками на macOS или Linux, рано или поздно столкнётесь с проблемой окончаний строк. Git показывает, что файлы изменились, хотя вы ничего не меняли. Это классическая проблема CRLF vs LF, и `core.autocrlf` — её решение.

## Проблема: LF vs CRLF

Разные операционные системы используют разные символы для обозначения конца строки:

**LF (Line Feed, `\n`)** — Unix/Linux/macOS. Один символ (байт 0x0A).

**CRLF (Carriage Return + Line Feed, `\r\n`)** — Windows. Два символа (байты 0x0D 0x0A).

**Проблема на практике:**

```
# Windows разработчик редактирует файл
# Редактор Notepad/VS Code сохраняет с CRLF: Строка\r\n

# Linux разработчик открывает тот же файл
# Git видит, что каждая строка "изменилась" (добавился \r)
git diff file.txt
# Показывает весь файл как изменённый, хотя содержимое то же
```

В результате: PR с тысячами «изменений» при открытии файла, конфликты при слиянии, сложно найти реальные изменения в истории.

## Решение: параметр core.autocrlf

`core.autocrlf` автоматически преобразует окончания строк при добавлении файлов в индекс и при получении файлов из репозитория.

**На Windows (рекомендуется):**

```bash
git config --global core.autocrlf true
```

Что это делает: при `git add` — преобразует CRLF в LF для репозитория; при `git checkout`/`git clone` — преобразует LF обратно в CRLF для рабочей директории.

**На Linux/macOS (рекомендуется):**

```bash
git config --global core.autocrlf input
```

Что это делает: при `git add` — преобразует CRLF в LF, если вдруг файл с CRLF попал; при `git checkout` — не меняет ничего (оставляет LF).

**Отключить преобразование (не рекомендуется для смешанных команд):**

```bash
git config --global core.autocrlf false
```

## Схема работы преобразований

```
core.autocrlf true (Windows):
Репозиторий (LF) → [checkout] → Рабочая директория (CRLF)
Рабочая директория (CRLF) → [git add] → Индекс (LF)

core.autocrlf input (Linux/macOS):
Репозиторий (LF) → [checkout] → Рабочая директория (LF)
Рабочая директория (CRLF) → [git add] → Индекс (LF)
```

В обоих случаях в репозитории хранится LF — это важно для совместимости.

## Проверка текущей конфигурации

```bash
# Посмотреть значение
git config core.autocrlf

# Посмотреть, какой файл устанавливает параметр
git config --show-origin core.autocrlf
# Вывод: file:~/.gitconfig  true

# Проверить окончания строк в файлах
git ls-files --eol
# Пример вывода:
# i/lf w/crlf attr/ file.txt   ← в индексе LF, в рабочей CRLF (Windows)
# i/lf w/lf   attr/ script.sh  ← везде LF
```

## Как исправить существующий проект

Если проект уже содержит смешанные окончания:

```bash
# 1. Установить правильный autocrlf
git config --global core.autocrlf input  # или true на Windows

# 2. Удалить из индекса все файлы (не удаляет из рабочей директории)
git rm --cached -r .

# 3. Переиндексировать файлы с правильными окончаниями
git add -A

# 4. Создать коммит с нормализацией
git commit -m "chore: нормализация окончаний строк"

# 5. Запушить на сервер
git push
```

## Файл .gitattributes: более гибкий подход

Для профессиональных проектов лучше использовать `.gitattributes` — он работает независимо от локальных настроек `autocrlf` и применяется ко всем разработчикам:

```
# .gitattributes в корне проекта

# Автоматически определять текстовые файлы и нормализовать
* text=auto

# Явно указать текстовые файлы
*.txt  text
*.md   text
*.html text
*.css  text
*.js   text

# Требовать LF (Unix-стиль) для скриптов
*.sh   text eol=lf
*.py   text eol=lf

# Требовать CRLF для Windows-файлов
*.bat  text eol=crlf
*.cmd  text eol=crlf
*.ps1  text eol=crlf

# Бинарные файлы — не трогать
*.jpg  binary
*.png  binary
*.gif  binary
*.zip  binary
*.pdf  binary
```

Добавьте `.gitattributes` в репозиторий:

```bash
git add .gitattributes
git commit -m "chore: добавлен .gitattributes для управления окончаниями строк"
```

## Как проверить окончания строк в файле

```bash
# Linux/macOS — показать невидимые символы
cat -A file.txt
# Строки с CRLF будут иметь ^M перед $: Текст^M$
# Строки с LF только: Текст$

# С помощью Git
git ls-files --eol

# Проверить конкретный файл
git check-attr eol file.txt
```

## Практические сценарии

```bash
# Сценарий 1: Новый Windows-разработчик присоединяется к проекту
git config --global core.autocrlf true
git clone https://github.com/team/project.git
# Всё работает правильно из коробки

# Сценарий 2: Проблема с файлами уже в проекте
git status
# Много изменённых файлов, хотя вы ничего не меняли?

git diff file.js
# Если видите только ^M (CRLF символы) — это проблема autocrlf
# Решение: настроить autocrlf и переиндексировать

# Сценарий 3: Профессиональный проект с командой
cat > .gitattributes << 'EOF'
* text=auto
*.sh text eol=lf
*.bat text eol=crlf
*.jpg binary
EOF

git add .gitattributes
git commit -m "chore: Add .gitattributes"
```

## Часто задаваемые вопросы

**Почему нужен autocrlf, если все используют UTF-8?** CRLF/LF не связано с кодировкой (UTF-8, UTF-16 и т.д.) — это про символы окончания строки, которые одинаковые во всех кодировках.

**Можно ли использовать autocrlf=true на Linux?** Технически можно, но не нужно. Linux инструменты (grep, sed, awk) работают с LF и могут некорректно обрабатывать CRLF.

**Что будет, если забыть установить autocrlf?** На Windows редактор создаст файлы с CRLF, Git увидит их как «изменённые» по сравнению с LF в репозитории. Это создаст шум в диффах.

**Как узнать, какие окончания использует файл?** `git ls-files --eol` покажет состояние в индексе и в рабочей директории. `cat -A file.txt` покажет `^M` перед `$` для CRLF.

**Нужен ли .gitattributes если у всех установлен autocrlf?** `.gitattributes` надёжнее, потому что не зависит от локальных настроек разработчиков. Рекомендуется для командных проектов.

## Заключение

Проблемы с CRLF/LF — классическая «засада» при смешанных командах. Правильная настройка решает всё: на Windows `core.autocrlf true`, на Linux/macOS — `input`. Для новых командных проектов рекомендуется добавить `.gitattributes` — это надёжнее и не зависит от настроек каждого разработчика.

Подробнее о настройке Git в целом — в статье [установка и настройка Git]({{< relref "nastrojka-git" >}}). Про .gitignore для исключения файлов из репозитория — [полное руководство по .gitignore]({{< relref "gitignore" >}}).
