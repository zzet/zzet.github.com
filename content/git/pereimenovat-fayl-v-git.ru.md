---
title: "Как переименовать файл в Git: команда git mv и ручное переименование"
description: "Инструкция по переименованию файлов в Git. Команда git mv, ручное переименование с сохранением истории, переименование папок, учёт регистра."
date: 2026-02-23
lastmod: 2026-02-23
draft: false
slug: "pereimenovat-fayl-v-git"
keywords: ["как переименовать файл в git", "git mv", "git переименовать файл", "переименовать файл git команда", "git mv старый новый", "как переименовать в git"]
tags: ["git", "beginner"]
categories: ["git"]
---

Переименовать файл в Git можно через команду `git mv` или вручную. Рекомендуется использовать `git mv` — она автоматически регистрирует переименование в индексе. При ручном переименовании нужно выполнить дополнительные шаги.

## Способ 1: git mv (рекомендуется)

```bash
# Переименовать файл
git mv old-name.txt new-name.txt

# Переименовать файл в другую папку
git mv src/old-file.js src/components/new-file.js

# Проверить статус — Git видит переименование
git status
# Changes to be committed:
#   renamed:    old-name.txt -> new-name.txt

# Закоммитить
git commit -m "Rename old-name.txt to new-name.txt"
```

`git mv` делает три вещи: переименовывает файл на диске, убирает старое имя из индекса, добавляет новое имя в индекс.

## Способ 2: ручное переименование

Если переименовали файл через IDE или файловый менеджер:

```bash
# Переименовали файл вручную (например, через проводник)
# Теперь нужно сообщить Git

# Удалить старое имя из отслеживания
git rm old-name.txt

# Добавить новое имя
git add new-name.txt

# Git определит переименование автоматически
git status
# Changes to be committed:
#   renamed:    old-name.txt -> new-name.txt

git commit -m "Rename old-name.txt to new-name.txt"
```

Или через `git add -A` — он отслеживает переименования, удаления и новые файлы:

```bash
# После ручного переименования
git add -A

git status
# renamed:    old-name.txt -> new-name.txt
```

## Переименование папок

```bash
# Переименовать папку
git mv old-folder/ new-folder/

# Переименовать папку с вложенными файлами
git mv src/components/ src/ui/

# Проверить
git status
# renamed:    src/components/Button.js -> src/ui/Button.js
# renamed:    src/components/Input.js  -> src/ui/Input.js

git commit -m "Rename components/ to ui/"
```

## Переименование с учётом регистра

На macOS и Windows файловые системы нечувствительны к регистру по умолчанию. Обычное переименование может не сработать:

```bash
# Неправильно (на macOS/Windows ничего не произойдёт):
git mv README.md readme.md

# Правильно: через временный файл
git mv README.md README_temp.md
git mv README_temp.md readme.md
git commit -m "Rename README.md to readme.md"

# Или через настройку ignorecase:
git config core.ignorecase false
git mv README.md readme.md
git commit -m "Rename README.md to readme.md"
git config core.ignorecase true
```

## Просмотр истории переименованного файла

После переименования стандартный `git log` не покажет историю до переименования:

```bash
# Только история после переименования
git log new-name.txt

# С флагом --follow — вся история, включая до переименования
git log --follow new-name.txt

# Подробная история с изменениями
git log --follow -p new-name.txt
```

Флаг `--follow` важен для отслеживания полной истории файла через переименования.

## Отмена переименования

Если ещё не закоммитили:

```bash
# Вернуть файл к исходному имени
git mv new-name.txt old-name.txt

# Или сбросить изменения индекса
git restore --staged new-name.txt old-name.txt
git restore new-name.txt

# Просмотреть что будет отменено
git status
```

Если уже закоммитили:

```bash
# Переименовать обратно и создать новый коммит
git mv new-name.txt old-name.txt
git commit -m "Revert rename: restore old-name.txt"

# Или через git revert для публичной ветки
git revert HEAD
# Создаст новый коммит отменяющий переименование
```

## Переименование через GitHub UI

Можно переименовать файл прямо на GitHub:

```
1. Открыть файл в GitHub
2. Нажать кнопку "Edit" (карандаш)
3. Изменить имя файла в поле вверху
4. Написать сообщение коммита "Rename filename.txt to newname.txt"
5. Нажать "Commit changes"
```

Минус этого способа: на GitHub это выглядит как удаление + создание нового файла (теряется атрибут rename). История может быть слегка нарушена.

## Переименование в VS Code и IntelliJ

**VS Code:**
- Кликнуть правой кнопкой на файл в Explorer → "Rename"
- Git автоматически зафиксирует переименование (если сделать git add)

**IntelliJ IDEA / PyCharm:**
- Shift+F6 (Refactor → Rename) на выделенном файле
- IDE сама создаст git mv команду и закоммитит

**Примечание:** IDE могут добавить дополнительные изменения кода если используют "refactoring rename" (переименование переменных, импортов и т.д.). Проверьте `git status` перед коммитом.

## Проблемы при переименовании на нечувствительных файловых системах

Windows и macOS по умолчанию используют нечувствительные к регистру файловые системы (NTFS, APFS). Это может вызвать проблемы:

```bash
# Нужно переименовать README.md → readme.md
# Прямое переименование не сработает:
git mv README.md readme.md
# На macOS/Windows файл может остаться как README.md

# Правильное решение: через временный файл
git mv README.md readme_temp.md
git mv readme_temp.md readme.md
git commit -m "Rename README.md to readme.md"

# Или отключить нечувствительность для этой операции
git config core.ignorecase false
git mv README.md readme.md
git config core.ignorecase true
git commit -m "Rename README.md to readme.md"
```

## Batch переименование (переименование множества файлов)

```bash
# Переименовать все .jsx файлы в .tsx
for f in *.jsx; do git mv "$f" "${f%.jsx}.tsx"; done
git commit -m "Rename .jsx files to .tsx"

# Переименовать все файлы из папки src/api в src/services
for f in src/api/*.js; do
  git mv "$f" "src/services/$(basename "$f")"
done
git commit -m "Move API files to services directory"

# Переименовать с префиксом
for f in *.ts; do git mv "$f" "old_$f"; done
git commit -m "Add old_ prefix to TypeScript files"

# С использованием find команды
find src -name "*.test.js" | while read f; do
  git mv "$f" "${f%.test.js}.spec.js"
done
git commit -m "Rename .test.js to .spec.js"
```

## Переименование нескольких файлов

```bash
# Переименовать несколько файлов по маске (через цикл)
for f in *.jsx; do git mv "$f" "${f%.jsx}.tsx"; done
git commit -m "Rename .jsx files to .tsx"

# Переименовать с изменением структуры
git mv src/api/users.js src/services/userService.js
git mv src/api/auth.js src/services/authService.js
git commit -m "Move API files to services directory"
```

## Часто задаваемые вопросы

**Сохраняется ли история файла после git mv?** Да, полностью. Для просмотра полной истории включая переименования используйте `git log --follow filename`. Без `--follow` `git log` покажет только историю после переименования. Флаг `--follow` работает только в `git log`, не в других командах.

**Чем git mv отличается от обычного mv?** `git mv` = `mv` + `git rm` + `git add` в одной команде. `git mv` сразу регистрирует переименование в индексе (stage). При обычном `mv` нужно вручную выполнить `git rm old` и `git add new`. Git всё равно распознает переименование через `git add -A`, но явное `git mv` чище.

**Может ли Git не определить переименование?** Да, если содержимое файла значительно изменилось. Git определяет переименование если совпадает >50% строк. Если это не работает, используйте `-M` флаг: `git diff -M50%` или `git diff -M80%`.

**Что такое git mv -f?** Флаг `-f` (force) перезаписывает файл если целевой файл уже существует. Используется редко, в основном в скриптах для автоматизации.

**Что такое git mv -k?** Флаг `-k` пропускает ошибки при переименовании множества файлов. Полезен при batch-переименовании через цикл где некоторые файлы могут не существовать.

**Как переименовать ветку?** Это другая команда: `git branch -m old-name new-name` для локальной ветки. `git branch -m` работает только с ветками, не с файлами. Для удалённых веток: `git push origin -u new-name origin/old-name` затем `git push origin :old-name` для удаления старой.

**Почему в GitHub показывается что файл удалён и создан новый, а не переименован?** Это может быть если переименование не было зарегистрировано правильно в commit'е. GitHub использует собственную логику определения переименований. Обычно помогает `git mv` вместо ручного переименования.

**Как переименовать файл сохранив содержимое?** `git mv` сохраняет всё содержимое полностью. Если вручную переименовали (через GUI или `mv`) и добавили в git — содержимое также сохранится. История правда может выглядеть как delete+add без `--follow`.

## Переименование с git rm и git add

Если забыли использовать `git mv` и уже переименовали файл через файловый менеджер:

```bash
# Файл переименован вручную но Git ещё не знает
# my-file.js → my-file.ts

git status
# deleted:       my-file.js
# Untracked files:
#   my-file.ts

# Решение 1: явно сообщить Git о переименовании
git rm my-file.js
git add my-file.ts

git status
# renamed:       my-file.js -> my-file.ts

# Решение 2: использовать git add -A
git add -A
git status
# renamed:       my-file.js -> my-file.ts
# Git сам определит переименование
```

## Отслеживание частичного переименования

Если файл изменился значительно и Git не определит переименование:

```bash
# Проверить как Git видит изменение
git diff --stat
# my-file.js => my-file.ts | 50 ++++++

# Если Git показывает как delete + add вместо rename:
# Это значит что совпадает менее 50% строк

# Для принудительного определения переименования
git diff -M50% --stat
# Теперь может показать как rename

# Если нужна точная история через переименование и изменения
git log --follow -p my-file.ts
```

## История переименования в GitHub

Когда просматриваете файл на GitHub:

```
File history:
- Rename my-file.js to my-file.ts (by John Doe)
- Update dependencies (by Jane Smith)
- Initial commit (by John Doe)
```

GitHub автоматически показывает переименования если они правильно зарегистрированы в Git. Это требует использования `git mv` или явного `git rm` + `git add`.

## Переименование и конфликты merge

Если в одной ветке файл переименовали, в другой его отредактировали:

```bash
# feature1: переименовал auth.js → authentication.js
# feature2: отредактировал auth.js → добавил новую функцию

# При слиянии feature1 в main потом feature2
git merge feature2
# CONFLICT (delete/modify): auth.js deleted in feature1...

# Git не знает что делать: файл удалён или переименован?

# Решение: явно указать что это переименование
git add src/authentication.js
git rm auth.js
git commit -m "Resolve rename conflict"
```

## Переименование в истории (git rebase)

Если нужно переименовать файл который был в старом коммите:

```bash
# Редактировать историю последних 5 коммитов
git rebase -i HEAD~5

# В редакторе заменить 'pick' на 'edit' для нужного коммита
# edit abc1234 feat: add auth

# Git остановится на этом коммите
# Выполнить переименование
git mv old-name.js new-name.js

# Обновить индекс и продолжить
git add new-name.js
git rm old-name.js
git commit --amend --no-edit
git rebase --continue

# История обновлена: в старом коммите файл уже с новым именем
```

## Синхронизация переименования с удалённым репо

После переименования файла и коммита:

```bash
# Локально всё готово
git log --oneline -1
# abc1234 Rename my-file.js to my-file.ts

# Отправить на сервер
git push origin feature-branch

# На удалённом репо GitHub автоматически определит переименование
# Если запушить с --force-with-lease после переименования в истории
git push --force-with-lease origin feature-branch
```

## Заключение

Используйте `git mv old-name new-name` для переименования файлов в Git — это самый чистый способ и гарантирует правильное определение переименования. После переименования запустите `git log --follow new-name` для просмотра полной истории. На системах с нечувствительной к регистру файловой системой (macOS, Windows) используйте временный файл или `core.ignorecase` для смены регистра. Для понимания состояния файлов после переименования — [git status]({{< relref "git-status" >}}).
