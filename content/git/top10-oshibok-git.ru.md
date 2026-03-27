---
title: "10 частых ошибок в Git и как их избежать"
description: "Топ-10 ошибок в Git: плохие коммиты, работа в main, игнорирование .gitignore, опасный force push. Как исправить и предотвратить каждую ошибку."
date: 2026-03-07
lastmod: 2026-03-07
draft: false
slug: "top10-oshibok-git"
keywords: ["ошибки git", "частые ошибки git", "как не делать ошибки git", "топ ошибок git", "как избежать ошибок в git", "git ошибки новичков"]
tags: ["git", "beginner", "intermediate"]
categories: ["git"]
---

Git прощает многие ошибки — большинство можно отменить. Но некоторые ошибки дорого обходятся команде. Вот 10 наиболее распространённых, с объяснением почему они происходят и как их исправить.

## 1. Забыли сделать git add для новых файлов

```bash
# Ошибка: git commit -a не добавляет новые файлы
touch new-feature.js
git commit -am "Add new feature"
# new-feature.js НЕ попал в коммит!

git status
# Untracked files:
#   new-feature.js

# Исправление:
git add new-feature.js
git commit --amend --no-edit  # добавить в последний коммит
# или
git commit -m "Add new-feature.js (forgot in previous commit)"
```

**Профилактика**: всегда проверяйте `git status` перед `git commit`.

## 2. Плохие сообщения коммитов

```bash
# Плохо:
git commit -m "fix"
git commit -m "update"
git commit -m "wip"
git commit -m "asdfgh"
git commit -m "изменения"

# Хорошо:
git commit -m "fix: resolve null pointer in user login"
git commit -m "feat: add password strength indicator"
git commit -m "docs: update API authentication guide"
```

**Почему важно**: через 6 месяцев `git log` с плохими сообщениями — бесполезен. Невозможно найти когда и почему был внесён конкретный баг.

**Исправление**: `git commit --amend -m "new message"` для последнего коммита. Для предыдущих — interactive rebase.

## 3. Слишком большие коммиты

```bash
# Плохо: один коммит с 20 несвязанными изменениями
git add -A
git commit -m "Various updates"  # изменено 15 файлов в 5 разных фичах

# Хорошо: отдельный коммит для каждого логического изменения
git add src/auth.js src/auth.test.js
git commit -m "feat: implement JWT authentication"

git add src/profile.js src/profile.test.js
git commit -m "feat: add user profile editing"
```

**Почему важно**: при бисекции (git bisect) или revert, маленькие коммиты позволяют точно найти проблему.

## 4. Работа напрямую в main

```bash
# Плохо:
git switch main
git commit -am "feat: add experimental feature"
git push origin main

# Хорошо:
git switch -c feature/experimental-feature
git commit -am "feat: add experimental feature"
git push origin feature/experimental-feature
# → Pull Request → Review → Merge
```

**Почему важно**: прямые коммиты в main обходят code review, могут сломать production, усложняют откат.

## 5. Игнорирование .gitignore

```bash
# Распространённые проблемы:
git add .
# Случайно добавлены: node_modules/, .env, *.log, dist/

# Исправление: создать .gitignore до первого git add
cat > .gitignore << 'EOF'
node_modules/
.env
*.log
dist/
.DS_Store
*.swp
EOF

# Если уже закоммитили:
git rm --cached -r node_modules/
git rm --cached .env
git commit -m "Remove accidentally tracked files"
```

**Совет**: используйте gitignore.io для генерации по языку/фреймворку.

## 6. Force push в общие ветки

```bash
# Опасно:
git push --force origin main  # НИКОГДА не делайте это в main!

# Что происходит:
# - Коллеги теряют коммиты из своих локальных копий
# - История расходится
# - Данные могут быть безвозвратно потеряны

# Безопаснее: --force-with-lease
git push --force-with-lease origin feature/my-branch
# Отклонит push если remote ветка изменилась (защита от перезаписи чужих коммитов)
```

**Правило**: force push допустим только в вашей личной feature ветке (которую никто не использует). В main — никогда.

## 7. Большие бинарные файлы в репозитории

```bash
# Плохо: коммит больших файлов
git add video.mp4        # 200 МБ
git add database.sql     # 50 МБ dump
git add design.psd       # 100 МБ

# Git репозиторий разрастается и клонирование становится медленным

# Решение 1: .gitignore для больших файлов
echo "*.sql" >> .gitignore
echo "*.psd" >> .gitignore

# Решение 2: Git LFS для нужных больших файлов
git lfs install
git lfs track "*.mp4"
git lfs track "*.psd"
git add .gitattributes
```

**Если уже закоммитили**: `git filter-repo --path-glob '*.sql' --invert-paths`

## 8. Merge вместо rebase для обновления ветки

```bash
# Результат: засорённая история с merge коммитами
# "Merge branch 'main' into feature/my-feature" повторяется 10 раз

# Лучше: rebase для обновления feature ветки
git fetch origin
git rebase origin/main

# Чистая линейная история
git log --oneline
# a1b2c3d feat: add user profile
# c3d4e5f feat: implement JWT auth
# ... (далее коммиты main)
```

**Правило**: rebase для обновления вашей feature ветки, merge для финального слияния в main.

## 9. Не использовать git stash или worktree

```bash
# Ошибка: переключение веток с незакоммиченными изменениями
git switch main
# error: Your local changes to the following files would be overwritten

# Или хуже: запустить git switch -f и потерять изменения

# Правильно: сохранить изменения перед переключением
git stash push -m "WIP: user profile form"
git switch main
# ... исправить баг ...
git switch feature/user-profile
git stash pop
```

## 10. Путаница reset и revert

```bash
# git reset — переписывает историю (опасно для shared branches)
git reset --hard HEAD~2  # УДАЛЯЕТ 2 последних коммита

# git revert — создаёт новый коммит-отмену (безопасно)
git revert HEAD          # создаёт коммит отменяющий последний

# Правило:
# - reset: только для локальных, ещё не запушенных коммитов
# - revert: для коммитов которые уже на remote

# Если случайно сделали reset — восстановить через reflog
git reflog
git reset --hard HEAD@{2}  # вернуться к состоянию 2 операции назад
```

## Общие принципы избежания ошибок

```bash
# 1. Всегда проверять статус перед коммитом
git status
git diff --staged

# 2. Использовать dry-run где возможно
git add --dry-run -A
git rm --dry-run *.log

# 3. Атомарные коммиты
# Один коммит = одно логическое изменение

# 4. Регулярно пушить feature ветки (backup)
git push origin feature/my-feature

# 5. Настроить алиасы для безопасности
git config --global alias.pushfl 'push --force-with-lease'

# 6. Изучить git reflog как инструмент восстановления
git reflog  # история всех операций HEAD
```

## Часто задаваемые вопросы

**Можно ли полностью удалить файл из истории?** Да, через `git filter-repo --path file.txt --invert-paths`, но это требует force push и пересинхронизации всей команды. Смените скомпрометированные секреты немедленно.

**Как предотвратить случайные коммиты секретов?** Используйте pre-commit хуки (detect-secrets пакет), `.gitignore` для `.env` файлов, и git-secrets инструмент.

**Что делать если запушил что-то плохое?** Не паникуйте. Если секреты — немедленно смените их (они уже скомпрометированы). Затем `git filter-repo` для очистки истории. Если просто плохой коммит — `git revert` или force push с `--force-with-lease` если ветка личная.

## Заключение

Большинство Git ошибок можно исправить через `git reflog`, `git revert` или `git reset`. Ключевые правила: маленькие осмысленные коммиты, никогда не force push в main, используйте `.gitignore` с самого начала, rebase вместо merge для обновления feature веток. Подробнее об отмене изменений — [git revert]({{< relref "git-revert" >}}) и [git reflog]({{< relref "git-reflog" >}}).

## По теме
- [git branch не показывает ветки]({{< relref "git-branch-ne-pokazyvaet" >}})
- [Поиск бага в истории: git bisect]({{< relref "git-bisect" >}})

- [Ошибка: untracked files]({{< relref "oshibka-untracked-files" >}})
