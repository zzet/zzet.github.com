---
title: "Протоколы передачи данных Git: HTTP, SSH, git:// и локальный"
description: "Протоколы Git для клонирования и синхронизации репозиториев: HTTPS, SSH, git://, локальный. Сравнение, настройка, когда что использовать."
date: 2026-02-27
lastmod: 2026-02-27
draft: false
slug: "protokoly-git"
keywords: ["протоколы git", "git ssh vs https", "git clone протокол", "git http протокол", "git ssh протокол", "какой протокол использовать git"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Git поддерживает четыре протокола для передачи данных между репозиториями: локальный файловый, HTTP/HTTPS, SSH и git://. Каждый имеет свои преимущества, ограничения и типичные случаи использования. Протокол определяет URL репозитория при клонировании.

## HTTPS протокол

HTTPS — самый универсальный протокол. Работает везде, где есть доступ к интернету, не требует настройки на клиенте:

```bash
# Клонирование через HTTPS
git clone https://github.com/user/project.git
git clone https://gitlab.com/user/project.git

# Push требует аутентификации
git push origin main
# Username for 'https://github.com': user
# Password for 'https://user@github.com': <токен>

# Кешировать учётные данные (не хранить в файле)
git config --global credential.helper cache
git config --global credential.helper 'cache --timeout=3600'

# Хранить в системном хранилище паролей
git config --global credential.helper osxkeychain  # macOS
git config --global credential.helper manager       # Windows
git config --global credential.helper libsecret     # Linux GNOME
```

GitHub и GitLab больше не принимают пароли — только Personal Access Tokens. Token используется вместо пароля при аутентификации.

## SSH протокол

SSH — предпочтительный протокол для разработчиков. Аутентификация через ключи, не требует ввода пароля при каждой операции:

```bash
# Клонирование через SSH
git clone git@github.com:user/project.git
git clone git@gitlab.com:user/project.git
git clone ssh://git@github.com/user/project.git  # альтернативный формат

# Настройка SSH ключа
ssh-keygen -t ed25519 -C "your@email.com"
# Публичный ключ: ~/.ssh/id_ed25519.pub
# Добавить в GitHub: Settings → SSH and GPG keys → New SSH key

# Проверить подключение
ssh -T git@github.com
# Hi username! You've successfully authenticated...

# Несколько аккаунтов: настройка ~/.ssh/config
cat >> ~/.ssh/config << 'EOF'
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
EOF

# Использовать разные аккаунты
git clone git@github-work:company/project.git
git clone git@github-personal:myuser/personal.git
```

## Локальный протокол

Для репозиториев на том же компьютере или в локальной сети:

```bash
# Клонировать по пути
git clone /path/to/repo.git
git clone /srv/git/repos/project.git

# Или с явным указанием протокола
git clone file:///path/to/repo.git

# Разница file:// vs путь:
# Путь без file:// — жёсткие ссылки (быстрее, меньше места)
# file:// — полное копирование объектов

# Использовать shared репозиторий в локальной сети
git clone //server/share/repos/project.git  # Windows UNC
git clone /mnt/nas/repos/project.git        # смонтированный NFS
```

## git:// протокол (git daemon)

Специальный протокол Git. Быстрый, но без аутентификации — только для чтения публичных репозиториев:

```bash
# Клонирование через git://
git clone git://github.com/user/project.git  # исторически, сейчас редко

# Запуск git daemon локально
git daemon \
  --reuseaddr \
  --base-path=/srv/git \
  --export-all \
  /srv/git
# Порт по умолчанию: 9418

# Клонировать с локального daemon
git clone git://localhost/project.git

# Разрешить конкретный репозиторий
touch /srv/git/project.git/git-daemon-export-ok
```

## Сравнение протоколов

```
Протокол  Порт   Аутентификация  Скорость  Настройка  Использование
HTTPS     443    Пароль/Токен    Средняя   Минимум    Публичные, CI/CD
SSH       22     Ключи           Высокая   SSH ключи  Разработка
git://    9418   Нет             Высокая   git daemon Публичные (read-only)
Локальный -      ФС права        Высокая   Нет        Один компьютер
```

## Смена протокола в существующем репозитории

```bash
# Посмотреть текущий URL
git remote -v
# origin  https://github.com/user/project.git (fetch)
# origin  https://github.com/user/project.git (push)

# Сменить с HTTPS на SSH
git remote set-url origin git@github.com:user/project.git

# Проверить
git remote -v
# origin  git@github.com:user/project.git (fetch)
# origin  git@github.com:user/project.git (push)

# Сменить с SSH на HTTPS
git remote set-url origin https://github.com/user/project.git
```

## Проксирование

```bash
# HTTPS через HTTP прокси
git config --global http.proxy http://proxy.example.com:8080
git config --global https.proxy https://proxy.example.com:8080

# SSH через HTTP прокси (nc или connect-proxy)
cat >> ~/.ssh/config << 'EOF'
Host github.com
    ProxyCommand nc -X connect -x proxy.example.com:8080 %h %p
EOF

# Отключить прокси для конкретного хоста
git config --global http.https://internal.server.com.proxy ""
```

## Настройка TLS/SSL

```bash
# Использовать кастомный CA сертификат
git config --global http.sslCAInfo /path/to/ca.crt

# Отключить проверку сертификата (НЕ рекомендуется для продакшена)
git config --global http.sslVerify false
# Для конкретного репозитория
git -c http.sslVerify=false clone https://internal.server/repo.git

# Указать сертификат клиента
git config --global http.sslCert /path/to/client.crt
git config --global http.sslKey /path/to/client.key
```

## Smart vs Dumb HTTP

Git поддерживает два варианта HTTP:

```bash
# Dumb HTTP (только чтение, статические файлы)
# Требует git update-server-info на сервере
# URL: https://example.com/repo.git

# Smart HTTP (чтение и запись, требует CGI/WSGI)
# Поддерживает push и аутентификацию
# Используют GitHub, GitLab, Bitbucket

# Проверить тип
git ls-remote https://example.com/repo.git
```

## Часто задаваемые вопросы

**SSH или HTTPS — что выбрать?** SSH предпочтительнее для регулярной разработки — один раз настроил ключи и больше не вводишь пароли. HTTPS проще для одноразовых операций и CI/CD (токены в переменных окружения).

**Почему SSH работает медленнее HTTPS на некоторых серверах?** SSH имеет overhead на установку соединения. Для крупных переносов данных разница незначительна. HTTPS с HTTP/2 иногда быстрее для большого количества объектов.

**Как настроить SSH если стандартный порт 22 заблокирован?** GitHub и GitLab поддерживают SSH через порт 443: `ssh.github.com` на порту 443. Добавьте в `~/.ssh/config`: `Host github.com; Hostname ssh.github.com; Port 443`.

**Что делать если HTTPS не работает за корпоративным прокси?** Настройте `http.proxy` в git config. Или используйте SSH через прокси с ProxyCommand в `~/.ssh/config`.

**Можно ли использовать git:// для приватных репозиториев?** Нет. git:// протокол не поддерживает аутентификацию. Только SSH или HTTPS для приватных репозиториев.

## Заключение

Для ежедневной разработки используйте SSH — один раз настроили ключ и работаете без паролей. Для CI/CD и автоматизации — HTTPS с токенами. git:// протокол подходит только для публичных репозиториев только для чтения. Порт SSH можно поменять — подробнее о [портах Git]({{< relref "port-git" >}}).
