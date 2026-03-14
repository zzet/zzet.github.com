---
title: "Порты Git: SSH 22, HTTPS 443, git:// 9418"
description: "Какие порты использует Git для разных протоколов. SSH порт 22, HTTPS 443, git daemon 9418. Как работать если порты заблокированы."
date: 2026-02-27
lastmod: 2026-02-27
draft: false
slug: "port-git"
keywords: ["git порт", "git ssh порт 22", "git https порт 443", "git daemon порт 9418"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Git использует разные порты в зависимости от выбранного протокола. SSH использует порт 22, HTTPS — порт 443, собственный git:// протокол — порт 9418. Понимание портов помогает решать проблемы подключения в корпоративных сетях.

## Порты по протоколам

```
Протокол   Порт   URL формат
SSH        22     git@github.com:user/repo.git
HTTPS      443    https://github.com/user/repo.git
git://     9418   git://github.com/user/repo.git
Локальный  -      /path/to/repo.git или file:///path/to/repo
```

## SSH: порт 22

SSH — стандартный протокол для разработчиков:

```bash
# Стандартное подключение через порт 22
git clone git@github.com:user/project.git

# Проверить доступность порта 22
ssh -T git@github.com
# Hi username! You've successfully authenticated...

# Если порт 22 заблокирован, проверить:
nc -zv github.com 22
# Connection to github.com 22 port [tcp/ssh] succeeded!

# Тест с подробным выводом
ssh -vT git@github.com 2>&1 | head -20
```

### SSH на нестандартном порту

GitHub и GitLab поддерживают SSH через порт 443 как обходной путь:

```bash
# GitHub: SSH через порт 443
ssh -T -p 443 ssh.github.com
# Hi username! You've successfully authenticated...

# Настроить в ~/.ssh/config для постоянного использования
cat >> ~/.ssh/config << 'EOF'
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
EOF

# Теперь стандартные команды работают через порт 443
git clone git@github.com:user/project.git

# GitLab: SSH через порт 443
cat >> ~/.ssh/config << 'EOF'
Host gitlab.com
    Hostname altssh.gitlab.com
    Port 443
    User git
EOF
```

### Собственный SSH сервер на другом порту

```bash
# Если ваш Git сервер использует нестандартный порт SSH
# Например, сервер на порту 2222

# ~/.ssh/config
Host myserver
    HostName server.example.com
    Port 2222
    User git

# Клонирование
git clone git@myserver:repos/project.git

# Или напрямую в URL
git clone ssh://git@server.example.com:2222/repos/project.git
```

## HTTPS: порт 443

HTTPS — универсальный протокол, работающий через любой firewall:

```bash
# Стандартное подключение через порт 443
git clone https://github.com/user/project.git

# Проверить доступность
curl -I https://github.com
# HTTP/2 200

# Если корпоративный прокси — настроить git
git config --global http.proxy http://proxy.corp.com:8080
git config --global https.proxy http://proxy.corp.com:8080

# Проверить прокси настройки
git config --global --get http.proxy
```

## git://: порт 9418

Специальный протокол Git для публичных репозиториев:

```bash
# Клонирование через git://
git clone git://github.com/user/project.git  # устарело на GitHub

# git daemon слушает на порту 9418
git daemon --reuseaddr --base-path=/srv/git --export-all /srv/git

# Проверить доступность
nc -zv server.example.com 9418
telnet server.example.com 9418

# Открыть порт в firewall (ufw)
sudo ufw allow 9418/tcp
```

Порт 9418 редко открыт в корпоративных сетях. GitHub прекратил поддержку git:// протокола.

## Диагностика проблем с портами

```bash
# Проверить открыт ли порт
nc -zv github.com 22   # SSH
nc -zv github.com 443  # HTTPS
nc -zv server.com 9418 # git://

# Трассировка соединения
traceroute github.com

# Проверить через curl (HTTPS)
curl -v https://github.com/user/project.git/info/refs?service=git-upload-pack 2>&1 | head -20

# SSH тест с подробным выводом
ssh -vvv git@github.com 2>&1 | grep -E "debug|Connecting|Connected|refused"

# Проверить текущие сетевые соединения git
GIT_TRACE=1 git fetch 2>&1 | head -20
```

## Работа в ограниченных сетях

```bash
# Корпоративная сеть с прокси: HTTPS + токен
git config --global http.proxy http://username:password@proxy:8080

# Если нужен токен для аутентификации через прокси
git config --global http.proxy http://proxy:8080
# Далее система паролей запросит учётные данные

# Только HTTPS через прокси — обойти SSH
# Настроить remote на HTTPS
git remote set-url origin https://github.com/user/project.git

# GitHub: если заблокированы все порты кроме 443
# Использовать SSH через HTTPS порт (как описано выше)
cat ~/.ssh/config
# Host github.com
#     Hostname ssh.github.com
#     Port 443
#     User git
```

## Какой порт использует конкретная команда

```bash
# Посмотреть фактически используемые соединения во время git операции
GIT_TRACE=1 GIT_TRACE_PACKET=1 git fetch 2>&1 | grep -i "connect\|port"

# strace для детального анализа (Linux)
strace -e connect git fetch 2>&1 | grep -E "connect|sin_port"

# Через ss: посмотреть активные соединения
ss -tnp | grep git
```

## Полная таблица протоколов и портов Git

Полный обзор всех способов подключения:

```
Протокол    Порт(ы)     URL формат                        Использование
─────────────────────────────────────────────────────────────────────────
SSH         22, 443     git@github.com:user/repo.git      Разработчики
HTTPS       443         https://github.com/user/repo.git  Всюду работает
git://      9418        git://github.com/user/repo.git    Редко (read-only)
HTTP        80          http://github.com/user/repo.git   Устарело
Файловая    -           /path/to/repo.git                 Локальные clone
            -           file:///path/to/repo.git          Через URL

Git Daemon  9418        Запускается с --base-path
SSH tunne   22+forwarding Проксирование через firewall
ProxyCommand Любой      Через jump host

─────────────────────────────────────────────────────────────────────────
```

## SSH config для удобного подключения

Настройка `~/.ssh/config` упрощает работу с несколькими портами и серверами:

```ini
# GitHub через обычный порт 22
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa

# GitHub через порт 443 (если 22 заблокирован)
Host github-443
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_rsa

# GitLab
Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_gitlab

# Собственный Git сервер
Host my-git-server
    HostName git.company.internal
    Port 2222
    User git
    IdentityFile ~/.ssh/company-key

# Использование:
# git clone git@github.com:user/repo.git           # через обычный порт
# git clone git@github-443:user/repo.git           # через порт 443
# git clone git@my-git-server:repos/project.git    # через 2222
```

## ProxyCommand для обхода firewall

Если корпоративный firewall блокирует большинство портов, можно использовать ProxyCommand:

```bash
# ~/.ssh/config
Host github.com
    HostName github.com
    User git
    ProxyCommand nc -X connect -x proxy.corp.com:8080 %h %p

# Для более сложных ситуаций с SSH сервером-прокси
Host github.com
    HostName github.com
    User git
    ProxyCommand ssh jumphost.corp.com -W github.com:22
```

Использование:

```bash
# Тестировать подключение
ssh -v git@github.com

# Clone работает через proxy
git clone git@github.com:user/repo.git
```

## Git Daemon: порт 9418 и безопасность

`git daemon` — встроенный сервер Git для анонимного доступа на чтение:

```bash
# Запустить git daemon
git daemon --reuseaddr --base-path=/srv/git --export-all --verbose

# С опциями безопасности (рекомендуется)
git daemon --reuseaddr \
  --base-path=/srv/git \
  --export-all \
  --user=git \
  --group=git \
  --detach

# Слушает на порту 9418
netstat -tlnp | grep 9418
# tcp  0  0 0.0.0.0:9418  0.0.0.0:*  LISTEN  1234/git-daemon
```

Клиенты подключаются как:

```bash
git clone git://server.example.com/project.git
```

Проблемы с git daemon:

```
⚠️ Нет аутентификации (кто угодно может клонировать)
⚠️ Нет шифрования (пароли видны в логах)
⚠️ Только чтение по умолчанию (нельзя делать push)
⚠️ Редко открыт в корпоративных сетях
⚠️ GitHub прекратил поддержку git:// в 2023 году
```

Когда использовать git daemon:

```
✓ Open source проект с публичным доступом (git read-only)
✓ Внутренний репозиторий в приватной сети
✓ Высокая нагрузка на читаемые операции
  (git daemon быстрее чем HTTP)
✓ Когда SSH/HTTPS недоступны
```

## HTTPS vs SSH: какой выбрать

Сравнение для разных сценариев:

```
Сценарий                    SSH             HTTPS
─────────────────────────────────────────────────────
Разработчик на ноутбуке     ✓ Лучше        ✓ Работает
                            (ключи)         (пароль)

CI/CD пайплайн              ✗ Сложнее      ✓ Проще
                            (ключи)         (токены)

Корпоративная сеть с proxy  ✗ Блокировано  ✓ Работает
                            (порт 22)       (порт 443)

Анонимный pull              ✗ Нет           ✓ Да
                            (нужна auth)    (no auth)

Быстрота                    ✓ Быстрее      ✓ Медленнее
                                           (TLS overhead)

Безопасность                ✓ SSH ключи    ✓ HTTPS TLS
                            ✓ Хорошая      ✓ Хорошая

─────────────────────────────────────────────────────
```

Рекомендация для разных команд:

```bash
# Разработчики: используйте SSH
git remote set-url origin git@github.com:user/repo.git

# CI/CD: используйте HTTPS с токеном
git clone https://username:$GITHUB_TOKEN@github.com/user/repo.git

# Fallback: если 22 закрыт, SSH через 443
# Настроить как описано выше в ~/.ssh/config
```

## Тестирование доступности портов

Полный набор команд для диагностики:

```bash
# 1. Проверить открыт ли конкретный порт (nc)
nc -zv github.com 22
nc -zv github.com 443
nc -zv github.com 9418

# 2. Альтернатива через telnet
telnet github.com 22
telnet github.com 443

# 3. Через SSH вербозно
ssh -v git@github.com
ssh -vvv git@github.com    # ещё более подробно

# 4. Проверить специфичный порт SSH с вербозом
ssh -v -p 443 git@ssh.github.com

# 5. Через curl (для HTTPS)
curl -v https://github.com/user/repo.git/info/refs?service=git-upload-pack 2>&1 | head -20

# 6. Трассировка маршрута до хоста
traceroute github.com

# 7. Проверить firewall правила (Linux)
sudo iptables -L -n
sudo ufw status

# 8. Посмотреть что Git реально делает
GIT_TRACE=1 git fetch
GIT_SSH_COMMAND="ssh -vvv" git fetch
```

## SSH конфигурация для различных сценариев

**Несколько учётных записей GitHub:**

```ini
# ~/.ssh/config
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/personal_key

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/work_key

# Использование:
# git clone git@github.com:personal/project.git        # personal_key
# git clone git@github-work:company/project.git        # work_key
```

**SSH через HTTP proxy:**

```ini
Host github.com
    HostName github.com
    User git
    ProxyCommand corkscrew proxy.corp.com 8080 %h %p
```

**Использование SSH с passphrase:**

```bash
# Запустить ssh-agent чтобы не вводить passphrase каждый раз
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
# Введёте passphrase один раз за сессию

# Или использовать key without passphrase для CI/CD
ssh-keygen -f ~/.ssh/ci_key -N ""  # без passphrase
```

## Проверка текущей конфигурации

```bash
# Посмотреть какой протокол использует текущий remote
git remote -v
# origin  https://github.com/user/repo.git (fetch)
# origin  https://github.com/user/repo.git (push)

# Или проверить конфиг
git config --get remote.origin.url

# Посмотреть SSH config
cat ~/.ssh/config

# Проверить какие ключи доступны
ssh-add -l

# Тестировать GitHub через оба портала
ssh -T git@github.com          # Порт 22
ssh -T -p 443 git@ssh.github.com  # Порт 443
```

## Часто задаваемые вопросы

**Почему git push/pull работает по HTTP но не по SSH?** Порт 22 заблокирован firewall или прокси. Решение: переключиться на HTTPS (`git remote set-url origin https://...`) или настроить SSH через порт 443.

**Нужно ли открывать порт 9418 для работы с GitHub?** Нет. GitHub использует HTTPS (443) и SSH (22/443). Порт 9418 (git://) GitHub не поддерживает с 2023 года.

**Как проверить через какой порт идёт трафик Git?** `GIT_TRACE=1 git fetch` покажет URL и метод подключения. URL определяет порт: `git@` → SSH 22, `https://` → HTTPS 443.

**Можно ли запустить SSH сервер для Git на порту 443?** Технически — да. Но это конфликтует с HTTPS веб-сервером на том же IP. Лучше использовать отдельный IP или SNI (HTTPS через nginx с stream proxying).

**Почему компании блокируют порт 9418?** Нестандартный порт, используется редко, нет шифрования и аутентификации. Firewall политики часто блокируют всё нестандартное по умолчанию.

## Заключение

Git использует три основных порта: 22 для SSH, 443 для HTTPS, 9418 для git://. В корпоративных сетях часто заблокирован SSH (22) — используйте SSH через порт 443 (ssh.github.com:443) или переключитесь на HTTPS. Подробнее о [протоколах Git]({{< relref "protokoly-git" >}}).
