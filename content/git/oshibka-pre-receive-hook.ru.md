---
title: "Ошибка «pre-receive hook declined» в Git"
description: "Как исправить ошибку pre-receive hook declined при git push. Причины: branch protection, commit policy, секреты в коде, force push. Диагностика и решения."
date: 2024-12-08
lastmod: 2024-12-08
draft: false
slug: "oshibka-pre-receive-hook"
keywords: ["pre receive hook declined", "git pre receive hook declined", "remote rejected master master pre receive hook declined", "remote rejected pre receive hook declined"]
tags: ["git", "errors", "hooks", "intermediate"]
categories: ["git"]
aliases: []
---

## Введение

Ошибка **"pre-receive hook declined"** — одна из самых распространённых проблем при работе с Git на удалённых репозиториях. Она появляется, когда сервер (GitHub, GitLab, Gitea) отклоняет ваш push в момент получения изменений. В этой статье мы разберёмся, что означает эта ошибка, почему она возникает, и как её исправить.

## 1. Что такое pre-receive hook и что означает ошибка

### Определение pre-receive hook

**Pre-receive hook** — это серверный скрипт, который выполняется на стороне Git-сервера (GitHub, GitLab, Gitea, Gitolite и т.д.) в момент получения `git push`. Хук запускается *до* того, как изменения будут записаны в репозиторий, и может:

- Отклонить push (вернуть exit code != 0)
- Разрешить push (вернуть exit code = 0)

Это отличается от **client-side hooks** (pre-commit, commit-msg), которые выполняются на вашем локальном компьютере.

### Полный текст ошибки

Когда pre-receive hook срабатывает и отклоняет push, вы видите сообщение:

```
remote: error: GH006: Protected branch update failed
remote: error: At least 1 approving review is required by reviewers with write access.
To github.com:username/repo.git
 ! [remote rejected] main -> main (pre-receive hook declined)
error: failed to push some refs to 'github.com:username/repo.git'
```

Или другие варианты:

```
remote: GitLab: You are not allowed to push code to this branch.
To gitlab.com:group/project.git
 ! [remote rejected] master -> master (pre-receive hook declined)
```

```
remote: error: hook declined to update refs/heads/feature
 ! [remote rejected] feature -> feature (pre-receive hook declined)
```

Ключевая строка — `[remote rejected] ... (pre-receive hook declined)` — говорит, что сервер **отклонил** ваш push.

## 2. Типичные причины срабатывания pre-receive hook

### 2.1 Branch protection rules (защита веток)

Одна из главных причин — на репозитории установлены **rules для защиты главных веток** (main, master).

**GitHub** защищает ветки так:
- Требует pull request вместо прямого push
- Требует одобрения от reviewer
- Требует прохождения CI/CD checks
- Требует актуального кода (нет конфликтов)

Если вы пытаетесь сделать `git push origin main` напрямую, а на ветке `main` установлена защита, сервер отклонит push.

### 2.2 Commit message policy

GitLab, Gitea и другие сервисы могут проверять **формат commit message**:
- Conventional commits (feat:, fix:, docs: и т.д.)
- Присутствие JIRA-номера (PROJ-123)
- Минимальная длина сообщения
- Запрет на определённые слова

Если сообщение не соответствует политике, push отклоняется.

### 2.3 Запрет force push

На защищённых ветках обычно **запрещён force push** (`git push -f`). Если вы попробуете перезаписать историю коммитов, сервер отклонит операцию.

### 2.4 Secret scanning (обнаружение секретов)

GitHub и GitLab имеют встроенный **secret scanning**, который обнаруживает:
- Пароли и API ключи
- Токены доступа
- Приватные ключи SSH/GPG

Если в коде обнаружен секрет, push автоматически отклоняется.

### 2.5 Обязательные подписанные коммиты

Для повышения безопасности может быть установлено требование, что все коммиты должны быть подписаны GPG ключом. Неподписанные коммиты будут отклонены.

### 2.6 Требование code review

На некоторых ветках требуется **как минимум одно одобрение** от членов команды перед merge. Обычно это реализовано через pull request, но сервер проверяет это в pre-receive хуке.

## 3. Диагностика: читаем вывод git push

Первый шаг при получении ошибки — **внимательно прочитать полный вывод** команды `git push`:

```bash
$ git push origin main
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 321 bytes | 321.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (0/0), done.
remote: error: GH006: Protected branch update failed
remote: error: At least 1 approving review is required by reviewers with write access.
To github.com:myuser/myrepo.git
 ! [remote rejected] main -> main (pre-receive hook declined)
error: failed to push some refs to 'github.com:myuser/myrepo.git'
```

В строках, начинающихся с `remote:`, сервер объясняет, **почему он отклонил push**. Это и есть диагностика! В примере выше: требуется одобрение reviewer.

## 4. Решения по каждому типу причины

### 4.1 Branch protection: создание PR

Если ветка защищена, нужно создать **Pull Request** вместо прямого push:

```bash
# Вместо:
git push origin feature-branch

# Создаём PR через веб-интерфейс GitHub/GitLab или через CLI:
gh pr create --title "My feature" --body "Description"
# или
glab mr create --title "My feature" --description "Description"
```

### 4.2 Commit message: исправляем сообщение

Если проблема в формате commit message, исправляем последний коммит:

```bash
# Просматриваем текущее сообщение
git log -1 --pretty=%B

# Редактируем сообщение последнего коммита
git commit --amend -m "feat: add new feature"

# Пушим с флагом -f, если это ваша приватная ветка
git push -f origin feature-branch
```

Или используем interactive rebase для нескольких коммитов:

```bash
git rebase -i HEAD~3  # Редактируем последние 3 коммита
# Затем пушим
git push -f origin feature-branch
```

### 4.3 Secrets: удаляем секреты из истории

Если обнаружены секреты (пароли, токены), нужно:

1. **Немедленно** повернуть/отозвать этот секрет на сервисе
2. Удалить секрет из истории Git

Для удаления используем `git filter-repo`:

```bash
# Установка (если не установлен)
pip install git-filter-repo

# Удаляем файл, содержащий секреты
git filter-repo --invert-paths --path secrets.txt

# Или заменяем содержимое в файле
git filter-repo --path-glob='*.env' --contents-literal-replacement \
  --contents-literal-old 'API_KEY=sk-1234567890abcdef' \
  --contents-literal-new 'API_KEY=***REMOVED***'

# Пушим с флагом force
git push -f origin main
```

**Важно**: секреты, попавшие в GitHub, уже видны сервису. Они обнаружены и отозваны ботом GitHub, но истории доступа могут остаться. Лучше всегда использовать `.gitignore` для файлов с секретами.

### 4.4 Signed commits: настройка GPG

Если требуются подписанные коммиты, настраиваем GPG:

```bash
# Генерируем GPG ключ (если не создали ранее)
gpg --full-generate-key

# Получаем ID ключа
gpg --list-secret-keys --keyid-format=long

# Подрегистрируем ключ в Git
git config --global user.signingkey <GPG_KEY_ID>

# Включаем автоматическую подпись
git config --global commit.gpgsign true

# Теперь все коммиты будут подписаны автоматически
git commit -m "feat: add feature"

# Или указываем флаг вручную
git commit -S -m "feat: add feature"
```

Затем добавляем публичный GPG ключ в GitHub/GitLab:

```bash
# Экспортируем публичный ключ
gpg --armor --export <GPG_KEY_ID>

# Копируем вывод и добавляем в Settings → SSH and GPG keys
```

### 4.5 Force push на защищённых ветках

Если у вас есть права администратора и вам **действительно** нужен force push:

**GitHub:**
1. Перейдите в Settings → Branches
2. Выберите защищённую ветку
3. Отметьте "Allow force pushes" → "Anyone"

**GitLab:**
1. Перейдите в Settings → Protected branches
2. Выберите ветку
3. Измените "Allowed to force push" на нужную роль

```bash
# После разрешения force push
git push -f origin main
```

## 5. Создание собственных pre-receive hooks

Если вы используете **самостоятельно размещённый Git-сервер** (GitLab CE, Gitea, Gitolite), вы можете создавать собственные pre-receive хуки.

### Структура скрипта

Pre-receive хук — это обычный bash-скрипт, который читает stdin и проверяет:
- Какие ветки изменяются
- Какие коммиты добавляются
- Какие файлы меняются

Скрипт должен быть размещён в `.git/hooks/pre-receive` (для локального repo) или в папке хуков сервера.

### Пример: проверка формата commit message

```bash
#!/bin/bash
# .git/hooks/pre-receive

# Читаем stdin в формате: oldrev newrev refname
while read oldrev newrev refname; do
  # Проверяем, что это ветка (не tag)
  if [[ $refname =~ ^refs/heads/ ]]; then
    # Получаем список новых коммитов
    commits=$(git rev-list $oldrev..$newrev)

    for commit in $commits; do
      message=$(git log -1 --format=%B $commit)

      # Проверяем, что сообщение начинается с: feat:, fix:, docs: и т.д.
      if ! [[ $message =~ ^(feat|fix|docs|style|refactor|test|chore): ]]; then
        echo "error: Commit message must start with 'feat:', 'fix:', etc."
        exit 1  # Отклоняем push
      fi
    done
  fi
done

exit 0  # Разрешаем push
```

### Exit codes

- **Exit 0** — хук успешен, push разрешён
- **Exit 1** (или любое число != 0) — хук не пройден, push отклонён

Всё, что выводится в stdout и stderr, будет показано пользователю в строках `remote:`.

## 6. Практические советы

1. **Всегда читайте полный вывод `git push`** — сервер указывает причину отклонения
2. **Используйте `.gitignore`** для файлов с секретами (`.env`, `secrets.txt`)
3. **Настройте локальные pre-commit хуки** для проверки сообщений и файлов перед commit
4. **Используйте branches для фич** вместо прямого push в main
5. **Работайте через Pull Requests** — это лучшая практика для командной разработки

## Часто задаваемые вопросы

### Q: Что такое pre-receive hook?

**A:** Pre-receive hook — это серверный скрипт, который выполняется на Git-сервере при получении `git push`. Он может проверить изменения и отклонить push, если они не соответствуют политикам репозитория (например, branch protection, формат сообщений, наличие секретов).

### Q: Как исправить ошибку "pre-receive hook declined"?

**A:** Зависит от причины:
- **Branch protection**: создайте Pull Request вместо прямого push
- **Commit message**: исправьте сообщение через `git commit --amend`
- **Secrets**: удалите секреты через `git filter-repo` и повернуть токен
- **Signed commits**: настройте GPG и подписывайте коммиты
- **Force push**: используйте `git push -f` только на приватных ветках

### Q: Как отключить branch protection на GitHub?

**A:** Если у вас есть права администратора:
1. Перейдите в Settings репозитория
2. Выберите Branches (слева)
3. Найдите защищённую ветку
4. Нажмите Edit или удалите правило
5. Отключите "Require pull requests before merging"

### Q: Можно ли обойти pre-receive hook?

**A:**
- **Разработчик**: обычно нет (если только вы не администратор)
- **Администратор**: да, через Settings → Branches → "Allow force pushes" или удаление rules
- **Лучший подход**: следовать политикам репозитория вместо их обхода

### Q: Чем pre-receive hook отличается от pre-commit hook?

**A:**
- **pre-commit** — выполняется **локально** на вашем компьютере перед созданием коммита, может быть легко пропущен (`git commit --no-verify`)
- **pre-receive** — выполняется **на сервере** при получении push, невозможно пропустить

### Q: Как убрать секреты из истории Git?

**A:** Используйте `git filter-repo`:

```bash
pip install git-filter-repo
git filter-repo --path-glob='*.env' --contents-literal-replacement \
  --contents-literal-old 'PASSWORD=secret123' \
  --contents-literal-new 'PASSWORD=***REMOVED***'
git push -f origin main
```

Или для удаления целого файла:

```bash
git filter-repo --invert-paths --path .env
git push -f origin main
```

**Важно**: после удаления секретов из истории немедленно повернуть эти секреты на сервисах (API, базы данных, облако).

## Заключение

Ошибка "pre-receive hook declined" — это **защитный механизм**, предотвращающий попадание проблемного кода в репозиторий. Вместо попыток обхода, лучше:

1. Понять причину отклонения (читаем `remote:` строки)
2. Исправить проблему (branch protection → PR, секреты → удалить и т.д.)
3. Пушить повторно

Так код остаётся чистым, безопасным и командная разработка становится гладче.

## Смотрите также

- {{< relref "git-push" >}} — основы команды `git push`
- {{< relref "oshibka-failed-push" >}} — другие ошибки при push