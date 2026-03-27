---
title: ".gitattributes: настройка атрибутов файлов в Git репозитории"
description: ".gitattributes задаёт правила для файлов в Git: переводы строк (eol), Git LFS, merge=ours для lock-файлов, diff атрибуты. Примеры для веб-проекта."
date: 2025-12-23
lastmod: 2025-12-23
draft: false
slug: "gitattributes"
keywords: ["gitattributes что это", ".gitattributes", "git attributes", "gitattributes merge=ours", "gitattributes eol", "gitattributes lfs", "git attributes файл"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Файл `.gitattributes` — это конфигурационный файл Git, который задаёт правила обработки файлов на основе их имён. Он позволяет контролировать поведение Git при работе с конкретными типами файлов в вашем репозитории.

**Основное назначение:** `*.gitattributes` позволяет вам сказать Git, как именно нужно обрабатывать файлы, соответствующие определённым шаблонам.

### .gitattributes vs .gitignore

Новички часто путают эти два файла:

- **`.gitignore`** — говорит Git, какие файлы НЕ отслеживать и не включать в репозиторий (например, `node_modules/`, `*.log`)
- **`.gitattributes`** — говорит Git, КАК обрабатывать файлы, которые УЖЕ отслеживаются в репозитории (например, как нормализовать переводы строк, какой инструмент использовать для слияния, как показывать дифф)

### Что контролирует .gitattributes

- **Переводы строк (line endings)** — LF vs CRLF на разных платформах
- **Слияние файлов** — какой инструмент использовать для разрешения конфликтов
- **Дифф-представление** — как показывать различия в дифф'е
- **Git LFS** — какие файлы хранить в Git LFS вместо обычного репозитория
- **Архивы** — какие файлы исключить при создании архива (`git archive`)

## Синтаксис файла .gitattributes

Файл `.gitattributes` содержит простые строки, состоящие из паттерна и атрибутов:

```gitattributes
# Синтаксис:
<pattern> <attribute1> <attribute2> ...

# Примеры:
*.txt text eol=lf
*.jpg binary
*.lock merge=ours
src/**/*.py text diff=python
```

### Паттерны

Паттерны используют синтаксис glob (как в `.gitignore`):

```gitattributes
# Все файлы с расширением
*.js text eol=lf
*.py text eol=lf
*.md text eol=lf

# Все файлы в директории
docs/ text
build/ binary

# Конкретный файл
package-lock.json merge=ours

# Несколько расширений
*.{jpg,png,gif} binary

# Глубокие пути
src/**/*.ts text diff=typescript
node_modules/ -text -diff
```

### Комментарии

Строки, начинающиеся с `#`, игнорируются:

```gitattributes
# Это комментарий
*.js text eol=lf  # Inline comment тоже работает
```

## Управление переводами строк (самое частое применение)

Когда разработчики работают на разных платформах (Windows, macOS, Linux), возникают проблемы с переводами строк:

- **Linux/macOS** используют LF (Line Feed, `\n`)
- **Windows** использует CRLF (Carriage Return + Line Feed, `\r\n`)

Если не настроить `.gitattributes`, при коммите каждая строка может быть изменена, что создаёт ненужные различия в дифф'ах.

### Автоматическая нормализация

```gitattributes
# Автоматически определять, какой тип файла (text или binary)
* text=auto

# Для текстовых файлов, явно указать LF
*.js text eol=lf
*.py text eol=lf
*.md text eol=lf
*.sh text eol=lf
```

Атрибут `text=auto` говорит Git: "Если это текстовый файл, нормализуй его. Если binary, оставь как есть."

### Явное указание LF для конкретных файлов

```gitattributes
# Все JavaScript файлы — всегда LF
*.js text eol=lf

# Все Python файлы — всегда LF
*.py text eol=lf

# Shell скрипты — всегда LF
*.sh text eol=lf
```

### Явное указание CRLF (редко, но иногда нужно)

```gitattributes
# Batch файлы Windows — всегда CRLF
*.bat text eol=crlf
*.cmd text eol=crlf
```

### Как это работает

Когда вы устанавливаете `eol=lf`:

1. **При коммите** (staging) — Git конвертирует CRLF → LF в репозитории
2. **При checkout** — Git конвертирует обратно в LF (или CRLF в зависимости от платформы и настроек)
3. **В дифф'е** — Git показывает реальные различия, без "изменено каждую строку"

## Атрибут binary

Атрибут `binary` говорит Git, что это бинарный файл, и его не нужно обрабатывать как текст:

```gitattributes
# Изображения
*.jpg binary
*.png binary
*.gif binary
*.svg binary

# Видео
*.mp4 binary
*.mov binary
*.avi binary

# Архивы
*.zip binary
*.tar.gz binary
*.7z binary

# Исполняемые файлы
*.exe binary
*.dll binary
*.so binary

# Документы
*.pdf binary
*.docx binary
*.xlsx binary
```

Установка `binary` эквивалентна `-text -diff`, что означает:
- Не обрабатывать как текст (не менять переводы строк)
- Не показывать дифф (невозможно увидеть изменения в текстовом виде)

## Git LFS атрибуты

Git LFS (Large File Storage) позволяет хранить большие файлы вне основного репозитория. Когда вы используете `git lfs track`, он автоматически добавляет атрибуты в `.gitattributes`:

```bash
# Установить Git LFS для PSD файлов
git lfs track "*.psd"

# Git автоматически добавит в .gitattributes:
# *.psd filter=lfs diff=lfs merge=lfs -text
```

### Полный синтаксис LFS

```gitattributes
# Большие двоичные файлы
*.psd filter=lfs diff=lfs merge=lfs -text
*.ai filter=lfs diff=lfs merge=lfs -text

# Большие видеофайлы
*.mov filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text

# Другие большие файлы
*.zip filter=lfs diff=lfs merge=lfs -text
*.dmg filter=lfs diff=lfs merge=lfs -text
```

### Что означает `filter=lfs`

- `filter=lfs` — используй Git LFS фильтр для этого типа файла
- `diff=lfs` — показывай дифф для LFS файлов (показывает метаинформацию)
- `merge=lfs` — использовать LFS для слияния этих файлов
- `-text` — не обрабатывать как текст

## merge=ours — стратегия слияния

Атрибут `merge=ours` — очень полезный инструмент. Он говорит Git: "Если будет конфликт при слиянии, всегда используй НАШУ версию файла и игнорируй другую."

### Основное использование

```gitattributes
# Автоматически генерируемые файлы — всегда используй нашу версию
package-lock.json merge=ours
yarn.lock merge=ours

# Сгенерированные файлы
*.generated merge=ours
dist/* merge=ours

# Логи и временные файлы
CHANGELOG.md merge=ours
.env.lock merge=ours
```

### Как включить merge=ours

1. **Убедитесь, что инструмент зарегистрирован** (обычно уже включён по умолчанию):

```bash
git config --global merge.ours.driver true
```

2. **Добавьте в .gitattributes:**

```gitattributes
package-lock.json merge=ours
```

3. **Теперь при конфликте** Git автоматически выберет вашу версию:

```bash
$ git merge feature-branch
# Conflicted package-lock.json — ours version kept
```

### Практический пример

Представьте ситуацию:

```
main:         feature-branch:
┌─────────┐  ┌──────────┐
│ package │  │ package  │
│ -lock   │  │ -lock    │
│ v1      │  │ v2       │
└─────────┘  └──────────┘
     \          /
      git merge
```

Без `merge=ours` — Git просит вас вручную разрешить конфликт.

С `merge=ours`:

```gitattributes
package-lock.json merge=ours
```

Git автоматически выбирает версию из main (вашей ветки), и конфликта нет.

### Случаи использования merge=ours

- **Файлы lock** (`package-lock.json`, `yarn.lock`, `Gemfile.lock`) — их нужно коммитить, но конфликты в них решаются просто (всегда берём нашу версию)
- **Сгенерированные файлы** (`dist/`, `build/`) — не должны быть в репозитории, но если случайно туда попали, конфликты нужно решить одной командой
- **Версионные файлы** (`.bumpversion.cfg`, `version.txt`) — часто редактируются локально, конфликты при merge'е можно решить автоматически

## diff атрибуты — улучшение вывода diff

По умолчанию Git показывает дифф в текстовом формате. Но для некоторых типов файлов это неоптимально:

- Для Python файлов будет полезно видеть имя функции в заголовке блока
- Для Java кода нужно видеть имя метода
- Для документов можно использовать специальные инструменты для конвертации в текст

### Встроенные diff функции

```gitattributes
# Python — показывать имя функции в заголовке
*.py diff=python

# Java — показывать имя метода
*.java diff=java

# C# — показывать имя класса или метода
*.cs diff=csharp

# Objective-C
*.m diff=objc

# Perl
*.pl diff=perl

# Ruby
*.rb diff=ruby
```

### Как это выглядит

**Без diff=python:**

```diff
@@ -42,3 +50,10 @@ import os
   def some_function():
```

**С diff=python:**

```diff
@@ -42,3 +50,10 @@ def some_function():
   def some_function():
```

Видна функция, в контексте которой произошли изменения.

### Кастомный diff для бинарных файлов

Можно использовать утилиты для конвертации бинарных файлов в текст для дифф'а:

```gitattributes
# Microsoft Word документы — конвертировать в текст
*.docx diff=word

# Excel таблицы
*.xlsx diff=excel

# PDF файлы
*.pdf diff=pdf
```

Для этого нужно зарегистрировать custom textconv:

```bash
git config diff.word.textconv "python -m docx2txt"
git config diff.pdf.textconv "pdftotext -"
git config diff.excel.textconv "ssconvert -"
```

## export-ignore — управление git archive

Команда `git archive` создаёт архив из репозитория (используется при создании релизов). Атрибут `export-ignore` исключает файлы и директории из архива:

```gitattributes
# Исключить тестовые файлы из архива
test/ export-ignore
tests/ export-ignore
*.test.js export-ignore
*.spec.js export-ignore

# Исключить разработческие файлы
.github/ export-ignore
.eslintrc export-ignore
.prettierrc export-ignore
webpack.config.js export-ignore

# Исключить документацию для разработчиков
CONTRIBUTING.md export-ignore
DEVELOPMENT.md export-ignore
docs/ export-ignore

# Исключить конфиги CI
.gitlab-ci.yml export-ignore
.travis.yml export-ignore
.circleci/ export-ignore
```

### Пример использования

```bash
# Создать архив, исключив файлы с export-ignore
git archive --format=tar.gz HEAD -o project-v1.0.tar.gz

# В архиве НЕ будет:
# - test/ директории
# - .github/ директории
# - CONTRIBUTING.md и другие помеченные файлы
```

## Полный пример .gitattributes для веб-проекта

Вот готовый, приработанный файл `.gitattributes` для типичного веб-проекта:

```gitattributes
# Auto-detect text files and normalize line endings to LF
* text=auto

# Source code
*.js text eol=lf
*.jsx text eol=lf
*.ts text eol=lf
*.tsx text eol=lf
*.py text eol=lf
*.java text eol=lf
*.cs text eol=lf
*.cpp text eol=lf
*.c text eol=lf
*.h text eol=lf
*.go text eol=lf
*.rs text eol=lf
*.rb text eol=lf
*.php text eol=lf

# Config and documentation
*.md text eol=lf
*.json text eol=lf
*.yaml text eol=lf
*.yml text eol=lf
*.xml text eol=lf
*.toml text eol=lf
*.ini text eol=lf
*.conf text eol=lf
*.properties text eol=lf
*.gradle text eol=lf
.env* text eol=lf
.editorconfig text eol=lf
.gitignore text eol=lf
.gitattributes text eol=lf

# Shell scripts
*.sh text eol=lf
*.bash text eol=lf
.bashrc text eol=lf
.bash_profile text eol=lf

# Windows scripts
*.bat text eol=crlf
*.cmd text eol=crlf
*.ps1 text eol=crlf

# Diff diff attributes для лучшего вывода
*.py diff=python
*.js diff=javascript
*.java diff=java
*.cs diff=csharp
*.cpp diff=cpp
*.rb diff=ruby

# Binary files
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.svg binary
*.ico binary
*.webp binary

*.mp4 binary
*.mov binary
*.avi binary
*.webm binary

*.zip binary
*.tar binary
*.tar.gz binary
*.7z binary
*.rar binary

*.pdf binary
*.docx binary
*.xlsx binary
*.pptx binary
*.doc binary
*.xls binary
*.ppt binary

*.exe binary
*.dll binary
*.so binary
*.o binary
*.class binary
*.jar binary

# Git LFS для больших файлов
*.psd filter=lfs diff=lfs merge=lfs -text
*.ai filter=lfs diff=lfs merge=lfs -text

# Lock files — always use ours version on merge
package-lock.json merge=ours
yarn.lock merge=ours
Gemfile.lock merge=ours
Pipfile.lock merge=ours

# Generated files
dist/ export-ignore
build/ export-ignore
node_modules/ export-ignore
.tox/ export-ignore
__pycache__/ export-ignore

# Development files
.github/ export-ignore
.gitlab-ci.yml export-ignore
.travis.yml export-ignore
Makefile export-ignore
CONTRIBUTING.md export-ignore
DEVELOPMENT.md export-ignore
```

## Как проверить применяются ли атрибуты

Для проверки, какие атрибуты установлены для конкретного файла, используйте команду `git check-attr`:

```bash
# Проверить все атрибуты для файла
git check-attr -a src/main.js

# Результат:
# src/main.js: text: set
# src/main.js: eol: lf
# src/main.js: diff: javascript

# Проверить конкретный атрибут
git check-attr text package.json
# Результат: package.json: text: set

# Проверить несколько файлов
git check-attr text *.js

# Проверить атрибут для всех файлов в директории
git check-attr -a -- *.py
```

## Часто задаваемые вопросы (FAQ)

**В: В чём разница между .gitattributes и .gitignore?**

О:
- `.gitignore` — говорит Git, какие файлы НЕ отслеживать
- `.gitattributes` — говорит Git, КАК обрабатывать файлы, которые отслеживаются

Например: `*.o` в `.gitignore` исключит все `.o` файлы, а `*.o binary` в `.gitattributes` скажет Git, как обработать `.o` файлы, если они уже в репозитории.

**В: Где расположить файл .gitattributes?**

О: Расположите `.gitattributes` в корне репозитория и закоммитьте его в Git. Он должен быть в репозитории, чтобы все разработчики использовали одинаковые правила.

```bash
# Расположение
/project-root/.gitattributes

# Закоммитьте его
git add .gitattributes
git commit -m "docs: add gitattributes for consistent line endings"
```

**В: Что делает merge=ours?**

О: `merge=ours` говорит Git: "Если есть конфликт при слиянии, автоматически используй НАШУ версию файла (из текущей ветки) и отбрось версию из другой ветки."

Это полезно для файлов, которые регулярно конфликтуют, но разрешение конфликта очевидно (например, lock файлы).

**В: Как проверить, применяются ли атрибуты к моим файлам?**

О: Используйте `git check-attr`:

```bash
git check-attr -a src/main.js
git check-attr text package.json
git check-attr eol -- *.js
```

**В: Влияет ли .gitattributes на существующие коммиты или только на новые?**

О: `.gitattributes` влияет на то, как Git **обрабатывает** файлы при различных операциях (коммит, checkout, дифф), но не меняет сохранённые в репозитории данные.

Если вы добавили `.gitattributes` после того, как уже закоммитили файлы с неправильными переводами строк, вам нужно:

```bash
# 1. Добавить .gitattributes
# 2. Нормализовать всё рабочую директорию:
git add -A
git commit -m "Normalize line endings"
```

## Внутренние ссылки

Изучите связанные темы:
- {{< relref "git-autocrlf" >}} — альтернативный способ настройки переводов строк
- {{< relref "git-lfs" >}} — полное руководство по Git LFS
- {{< relref "gitignore" >}} — подробно о файле .gitignore

## Заключение

Файл `.gitattributes` — это мощный инструмент для стандартизации работы с файлами в репозитории. Используйте его для:

1. **Контроля переводов строк** — чтобы избежать "изменён каждую строку" в дифф'ах
2. **Настройки слияния** — используйте `merge=ours` для файлов, которые регулярно конфликтуют
3. **Улучшения дифф'ов** — укажите тип файла для лучшего отображения различий
4. **Управления архивами** — исключите файлы с `export-ignore`

Добавьте `.gitattributes` в корень репозитория, закоммитьте его, и все разработчики будут работать с одинаковыми правилами обработки файлов.

## По теме

- [Символические ссылки в Git]({{< relref "git-symlinks" >}})
