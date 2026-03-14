---
title: "Git и CRLF/LF: решение проблемы переносов строк в Windows"
description: "Как решить проблему CRLF vs LF в Git. Настройка core.autocrlf, .gitattributes, исправление файлов, конфигурация для Windows и macOS."
date: 2026-01-09
lastmod: 2026-01-09
draft: false
slug: "git-crlf-windows"
keywords: ["git crlf", "LF vs CRLF git", "переносы строк git windows", "git windows перенос строки", "crlf lf git config", "git eol windows"]
tags: ["git", "windows", "intermediate"]
categories: ["git"]
---

Проблема CRLF/LF — одна из самых распространённых при работе с Git на Windows. Windows использует CRLF (`\r\n`) для переносов строк, а Linux/macOS — LF (`\n`). Git может автоматически конвертировать — но важно правильно настроить.

## Что такое CRLF и LF

```
LF (Line Feed, \n, 0x0A):
- Используется в Linux, macOS, Unix
- Один символ для переноса строки

CRLF (Carriage Return + Line Feed, \r\n, 0x0D 0x0A):
- Используется в Windows
- Два символа для переноса строки

Проблема:
- Разработчик на Windows сохраняет файл с CRLF
- Git видит что файл изменился (хотя код не менялся)
- Разработчик на Linux получает файл с CRLF — некоторые инструменты не понимают
```

## Настройка core.autocrlf

`core.autocrlf` управляет автоматической конвертацией:

```bash
# Для Windows: конвертировать в LF при коммите, CRLF при checkout
git config --global core.autocrlf true

# Для Linux/macOS: конвертировать CRLF в LF при коммите (только)
git config --global core.autocrlf input

# Отключить автоконвертацию (не рекомендуется)
git config --global core.autocrlf false
```

**Поведение:**

```
core.autocrlf=true (Windows):
  commit: CRLF → LF (нормализация перед сохранением)
  checkout: LF → CRLF (Windows получает привычные окончания)

core.autocrlf=input (Linux/macOS):
  commit: CRLF → LF (исправление ошибок)
  checkout: без изменений

core.autocrlf=false:
  commit: без изменений
  checkout: без изменений
```

## .gitattributes — лучший способ

`.gitattributes` — более надёжный способ, работает для всей команды независимо от локальных настроек:

```bash
# Создать .gitattributes в корне репозитория
cat > .gitattributes << 'EOF'
# Нормализовать все текстовые файлы в LF
* text=auto

# Явно указать текстовые файлы
*.js text eol=lf
*.ts text eol=lf
*.jsx text eol=lf
*.tsx text eol=lf
*.py text eol=lf
*.rb text eol=lf
*.html text eol=lf
*.css text eol=lf
*.scss text eol=lf
*.json text eol=lf
*.md text eol=lf
*.yml text eol=lf
*.yaml text eol=lf
*.sh text eol=lf

# Бинарные файлы — без конвертации
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.ico binary
*.zip binary
*.tar.gz binary
*.pdf binary
*.exe binary
*.dll binary

# Файлы специфичные для Windows (CRLF)
*.bat text eol=crlf
*.cmd text eol=crlf
EOF

git add .gitattributes
git commit -m "Add .gitattributes for consistent line endings"
```

## Исправление файлов с неправильными окончаниями

Если в репозитории уже есть файлы с CRLF:

```bash
# Проверить какие файлы содержат CRLF
file * | grep CRLF
# или
grep -rl $'\r' . --include="*.js"

# Конвертировать файлы в LF
# Способ 1: dos2unix (установить отдельно)
dos2unix src/*.js

# Способ 2: через Git (после настройки .gitattributes)
git rm --cached -r .
git reset --hard

# Способ 3: sed
sed -i 's/\r//' filename.js

# После конвертации зафиксировать
git add .
git commit -m "Normalize line endings to LF"
```

## Предупреждение "CRLF will be replaced by LF"

Распространённое предупреждение при `git add` на Windows:

```
warning: CRLF will be replaced by LF in filename.js.
The file will have its original line endings in your working directory.
```

Это нормальное поведение при `core.autocrlf=true` — Git уведомляет что конвертирует файл перед сохранением в репозиторий. Можно подавить:

```bash
# Подавить предупреждение
git config --global core.safecrlf false

# Или сделать предупреждение ошибкой (строгий режим)
git config --global core.safecrlf true
```

## Настройка для VS Code

VS Code позволяет настроить окончания строк для конкретного проекта:

```json
// .vscode/settings.json
{
  "files.eol": "\n",
  "files.encoding": "utf8"
}
```

Это не заменяет `.gitattributes`, но устанавливает поведение редактора по умолчанию.

## Проверка окончаний строк файла

```bash
# Показать скрытые символы (включая \r)
cat -A filename.txt
# Строки с CRLF заканчиваются на ^M$
# Строки с LF заканчиваются на $

# Через hexdump
hexdump -C filename.txt | grep "0d 0a"
# 0d = \r (CR), 0a = \n (LF)

# file утилита
file filename.txt
# ASCII text  — Unix LF
# ASCII text, with CRLF line terminators — Windows CRLF
```

## Рекомендуемая конфигурация

```bash
# Windows разработчик:
git config --global core.autocrlf true

# Linux/macOS разработчик:
git config --global core.autocrlf input

# Для всей команды (лучший вариант):
# Создать .gitattributes в репозитории:
echo "* text=auto eol=lf" > .gitattributes
git add .gitattributes
git commit -m "Enforce LF line endings"
```

## Часто задаваемые вопросы

**Почему git status показывает изменения когда файл не менялся?** Скорее всего окончания строк изменились. Проверьте `core.autocrlf` и `file` для анализа типа окончаний.

**Какой стандарт выбрать — LF или CRLF?** В большинстве проектов — LF. Все современные редакторы на Windows (VS Code, Notepad++) прекрасно понимают LF. CRLF только для .bat/.cmd файлов.

**Почему .gitattributes лучше core.autocrlf?** `.gitattributes` хранится в репозитории и применяется для всей команды независимо от локальных настроек. `core.autocrlf` — локальная настройка, каждый разработчик должен настроить сам.

**Как исправить репозиторий с перемешанными LF/CRLF?** Добавьте `.gitattributes`, потом `git rm --cached -r . && git reset --hard` — перечитает все файлы с учётом новых правил. Зафиксируйте в коммит.

## Заключение

Проблема CRLF/LF в Git решается через `.gitattributes` (`* text=auto eol=lf`) — лучший и надёжный способ для команды. На Windows также используйте `core.autocrlf true`. В VS Code настройте `files.eol: "\n"`. Исправление уже испорченных файлов: `dos2unix` или пересоздание индекса через `git rm --cached -r .`. Подробнее о настройке Git — [настройка пользователя Git]({{< relref "nastrojka-polzovatelya-git" >}}).
