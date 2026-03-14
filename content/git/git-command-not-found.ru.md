---
title: "bash: git: command not found — причины и решение"
description: "Ошибка bash: git: command not found означает что Git не установлен или не добавлен в PATH. Решение для Windows, macOS, Linux и WSL."
date: 2026-01-08
lastmod: 2026-01-08
draft: false
slug: "git-command-not-found"
keywords: ["bash git command not found", "git command not found", "git не найден", "git не установлен", "git path не найден", "zsh git command not found"]
tags: ["git", "beginner", "troubleshooting"]
categories: ["git"]
---

## bash: git: command not found — причины и решение

Вы открыли терминал, набрали `git status` или `git --version` — и получили:

```
bash: git: command not found
```

или в zsh:

```
zsh: command not found: git
```

или в Windows PowerShell:

```
'git' is not recognized as an internal or external command
```

Ошибка означает одно из двух: либо Git не установлен, либо установлен, но система не знает где его искать (проблема с PATH). Разберём оба случая для каждой платформы.

---

## Шаг 1: проверить установлен ли Git

Сначала убедитесь, что Git вообще есть в системе:

```bash
# Linux / macOS / WSL
which git
# Если ничего не выведено — Git не установлен

# Или попробуйте полный путь
/usr/bin/git --version
/usr/local/bin/git --version
```

Если `which git` ничего не вернул — переходите к установке для вашей платформы. Если путь нашёлся (`/usr/bin/git`) — проблема в PATH, переходите к разделу про PATH.

---

## Решение на Linux

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install git

# Проверить
git --version
# git version 2.39.x
```

### Fedora / RHEL / CentOS

```bash
sudo dnf install git
# или для старых версий:
sudo yum install git

git --version
```

### Arch Linux

```bash
sudo pacman -S git
```

### Alpine Linux (Docker)

```bash
apk add git
```

После установки ошибка `command not found` должна исчезнуть — пакетный менеджер автоматически добавляет Git в `/usr/bin/git`, который всегда есть в PATH.

---

## Решение на macOS

### Вариант 1: Установить через Xcode Command Line Tools (рекомендуется)

```bash
xcode-select --install
```

Появится диалоговое окно — нажать «Установить». После завершения:

```bash
git --version
# git version 2.39.x (Apple Git)
```

### Вариант 2: Установить через Homebrew

```bash
# Установить Homebrew (если нет)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Установить Git
brew install git

git --version
# git version 2.43.x
```

Homebrew-версия Git новее чем Apple Git — рекомендуется для разработчиков.

### Если macOS предлагает установить Git при первом вызове

На свежем macOS при вводе `git` система сама предлагает установить Command Line Tools через диалог. Просто нажмите «Установить» — это быстро и бесплатно.

---

## Решение на Windows

### Если Git не установлен

Скачайте официальный установщик с [git-scm.com/download/win](https://git-scm.com/download/win) и установите его. Подробная инструкция: {{< relref "ustanovka-git-windows" >}}.

При установке обязательно выберите **«Git from the command line and also from 3rd-party software»** — это добавит Git в системный PATH.

### Если Git установлен, но не найден в PowerShell/CMD

Git скорее всего установлен только для Git Bash, но не добавлен в системный PATH.

**Способ 1: переустановить с правильной опцией PATH**

1. Скачать установщик заново
2. На шаге «Adjusting your PATH environment» выбрать «Git from the command line and also from 3rd-party software»
3. Завершить установку

**Способ 2: добавить Git в PATH вручную**

1. Найти где установлен Git: обычно `C:\Program Files\Git\bin`
2. Открыть: Пуск → Система → Дополнительные параметры системы → Переменные среды
3. В разделе «Системные переменные» найти `Path` → «Изменить»
4. Нажать «Создать» и добавить: `C:\Program Files\Git\bin`
5. Нажать «Создать» ещё раз и добавить: `C:\Program Files\Git\cmd`
6. ОК → ОК → Перезапустить терминал

```powershell
# Проверить после добавления в PATH
git --version
# git version 2.43.x.windows.1
```

**Способ 3: добавить через PowerShell**

```powershell
# Добавить Git в PATH через PowerShell (запустить от администратора)
$gitPath = "C:\Program Files\Git\bin"
[System.Environment]::SetEnvironmentVariable(
  "PATH",
  [System.Environment]::GetEnvironmentVariable("PATH", "Machine") + ";$gitPath",
  "Machine"
)
# Перезапустить PowerShell
```

---

## Решение в WSL (Windows Subsystem for Linux)

WSL — это Linux внутри Windows, и Git из Windows сюда не передаётся. Нужно установить Git отдельно внутри WSL:

```bash
# Внутри WSL (Ubuntu)
sudo apt update && sudo apt install git

git --version
# git version 2.39.x
```

Это нормальное поведение: WSL и Windows — разные окружения с разными PATH. Git нужно иметь в обоих если используете оба.

---

## Проблема с PATH: Git установлен, но не найден

Если `which git` нашёл путь, но обычный вызов `git` не работает — проблема в том что директория с Git не в переменной `PATH`.

### Проверить текущий PATH

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

Git обычно находится в `/usr/bin/git` или `/usr/local/bin/git`. Если соответствующая директория не в PATH — добавьте её:

```bash
# Добавить в ~/.bashrc (bash) или ~/.zshrc (zsh)
export PATH="/usr/local/bin:$PATH"

# Применить без перезапуска
source ~/.bashrc
# или
source ~/.zshrc
```

### Если Git установлен через Homebrew на macOS, но не найден

Homebrew на Apple Silicon (M1/M2/M3) устанавливает пакеты в `/opt/homebrew/bin`, а не в `/usr/local/bin`. Добавьте в `~/.zshrc`:

```bash
export PATH="/opt/homebrew/bin:$PATH"
source ~/.zshrc
```

### Если используете нестандартный shell

```bash
# Проверить текущий shell
echo $SHELL
# /bin/zsh  или  /bin/bash  или  /usr/bin/fish

# Для fish shell — конфиг в ~/.config/fish/config.fish
fish_add_path /usr/local/bin
```

---

## Ошибка в CI/CD (GitHub Actions, GitLab CI)

Если ошибка `command not found: git` появляется в пайплайне:

```yaml
# GitHub Actions: добавить шаг установки
- name: Install Git
  run: sudo apt-get install -y git

# Или использовать образ с предустановленным Git
- uses: actions/checkout@v4  # этот action сам использует git
```

В большинстве стандартных CI образов (`ubuntu-latest`, `ubuntu-22.04`) Git уже предустановлен. Если нет — добавьте явную установку.

---

## Ошибка в Docker контейнере

```dockerfile
# Добавить в Dockerfile
RUN apt-get update && apt-get install -y git

# Или для Alpine
RUN apk add git
```

---

## Быстрая диагностика: какая у вас ситуация

| Ситуация | Решение |
|----------|---------|
| macOS, `git` никогда не работал | `xcode-select --install` |
| macOS + Homebrew, git не найден | `export PATH="/opt/homebrew/bin:$PATH"` |
| Linux, чистая система | `sudo apt install git` или `sudo dnf install git` |
| Windows, Git не установлен | Скачать с git-scm.com, выбрать правильный PATH вариант |
| Windows, Git есть но не в CMD/PowerShell | Добавить `C:\Program Files\Git\bin` в системный PATH |
| WSL, `git` не найден | Установить git внутри WSL отдельно |
| Всё правильно, но всё равно не работает | Перезапустить терминал / переоткрыть сессию |

---

## FAQ

**Почему `git` работает в Git Bash, но не в PowerShell?**
Git Bash настраивает свой PATH автоматически при запуске. PowerShell использует системный PATH. Если при установке не была выбрана опция «Git from the command line», системный PATH не обновился. Переустановите Git с правильной опцией или добавьте путь вручную.

**После установки ошибка всё ещё есть — что делать?**
Перезапустите терминал. Изменения PATH применяются только в новых сессиях, не в уже открытых окнах.

**`git --version` работает, но `git status` не работает?**
Это уже другая ошибка — вы не в git репозитории или другая проблема. `command not found` означает именно что сама программа git не найдена.

**Как проверить что Git точно добавлен в PATH?**

```bash
which git        # Linux/macOS — покажет полный путь
where git        # Windows CMD — покажет все вхождения в PATH
Get-Command git  # Windows PowerShell
```

**Нужно ли устанавливать Git отдельно в каждом виртуальном окружении Python?**
Нет. Git — системная утилита, не Python пакет. Она устанавливается один раз на уровне ОС и доступна во всех окружениях.

---

## Итог

Ошибка `bash: git: command not found` решается в два шага: установить Git (`apt install git` / `brew install git` / официальный установщик на Windows) и убедиться что директория с Git есть в переменной PATH. Самая частая проблема на Windows — при установке не была выбрана опция добавления Git в системный PATH. Самая частая на macOS — Homebrew-Git установлен в `/opt/homebrew/bin`, но эта директория не в `$PATH`.

Подробная инструкция по установке Git на Windows: {{< relref "ustanovka-git-windows" >}}.
