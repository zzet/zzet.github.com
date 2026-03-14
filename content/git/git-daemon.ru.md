---
title: "git daemon: запуск Git сервера по протоколу git://"
description: "Как запустить git daemon для раздачи репозиториев по протоколу git://. Настройка, порты, systemd сервис, ограничения и безопасность."
date: 2026-01-10
lastmod: 2026-01-10
draft: false
slug: "git-daemon"
keywords: ["git daemon", "git daemon запуск", "git:// протокол сервер", "git daemon --base-path", "git:// сервер настройка", "git daemon Windows"]
tags: ["git", "advanced"]
categories: ["git"]
---

`git daemon` — встроенный в Git простой сервер, раздающий репозитории по протоколу `git://`. Он быстрый, не требует аутентификации и подходит для публичных репозиториев в локальной сети. Порт по умолчанию: 9418.

## Что такое git daemon

Git daemon работает по собственному протоколу `git://` (не HTTP, не SSH). Особенности:

- Только чтение (по умолчанию) — нельзя делать push
- Нет аутентификации — любой может клонировать
- Быстрый — минимальный overhead
- Порт 9418 — нестандартный, часто заблокирован на firewall
- Подходит для внутренних сетей, публичных зеркал

```bash
# Протокол git://
git clone git://server.example.com/project.git
git clone git://localhost/project.git
```

## Базовый запуск

```bash
# Самый простой запуск
git daemon --reuseaddr --base-path=/srv/git --export-all /srv/git

# Параметры:
# --reuseaddr     — переиспользовать адрес после рестарта
# --base-path=DIR — базовый путь для репозиториев
# --export-all    — раздавать все репозитории (иначе только с маркером)

# Клонировать
git clone git://server.example.com/project.git
# Путь: /srv/git/project.git → URL: git://server/project.git
```

Без `--export-all` нужно создавать маркер для каждого репозитория:

```bash
# Разрешить конкретный репозиторий (без --export-all)
touch /srv/git/project.git/git-daemon-export-ok

# Или запретить репозиторий когда используется --export-all
# (Нет встроенного механизма — используйте .git/daemon-deny или
#  не используйте --export-all)
```

## Настройка с опциями

```bash
# Полная настройка
git daemon \
  --reuseaddr \
  --verbose \
  --base-path=/srv/git \
  --export-all \
  --enable=receive-pack \
  --listen=0.0.0.0 \
  --port=9418 \
  /srv/git

# Разрешить push (--enable=receive-pack)
# ВНИМАНИЕ: позволяет ЛЮБОМУ делать push — только для доверенных сетей!

# Слушать только localhost (для тестирования)
git daemon --base-path=/srv/git --export-all --listen=127.0.0.1

# Раздавать конкретные директории
git daemon --base-path=/srv --export-all \
  /srv/git/public-repos
```

## Запуск как systemd сервис

Для постоянного запуска на сервере используйте systemd:

```bash
# Создать файл сервиса
sudo cat > /etc/systemd/system/git-daemon.service << 'EOF'
[Unit]
Description=Git Daemon
After=network.target

[Service]
User=git
Group=git
ExecStart=/usr/bin/git daemon \
    --reuseaddr \
    --verbose \
    --base-path=/srv/git \
    --export-all \
    /srv/git
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Включить и запустить
sudo systemctl daemon-reload
sudo systemctl enable git-daemon
sudo systemctl start git-daemon

# Статус
sudo systemctl status git-daemon
# ● git-daemon.service - Git Daemon
#      Active: active (running)
```

## Запуск через inetd

Альтернатива systemd — запуск через inetd (xinetd):

```bash
# /etc/inetd.conf
echo "9418 stream tcp nowait git /usr/bin/git \
  git daemon --inetd --verbose --export-all --base-path=/srv/git" \
  >> /etc/inetd.conf

# Перезапустить inetd
sudo service inetd restart
```

## Firewall и безопасность

```bash
# Открыть порт 9418 в firewall (ufw)
sudo ufw allow 9418/tcp
sudo ufw reload

# Iptables
sudo iptables -A INPUT -p tcp --dport 9418 -j ACCEPT

# Проверить что git daemon слушает
ss -tlnp | grep 9418
# LISTEN 0 128 0.0.0.0:9418 ...

# Тест подключения
git ls-remote git://localhost/project.git
```

Git daemon полностью открыт — нет аутентификации. Используйте только для:
- Публичных репозиториев
- Изолированных внутренних сетей
- Временных серверов для миграции данных

## Работа с git daemon для push

Push через git:// требует явного включения и крайне небезопасен без ACL:

```bash
# Включить receive-pack (push) — ТОЛЬКО в доверенной сети
git daemon \
  --base-path=/srv/git \
  --export-all \
  --enable=receive-pack \
  /srv/git

# Теперь можно делать push
git push git://server/project.git main

# Ограничить по IP через tcp wrappers /etc/hosts.allow
# git-daemon: 192.168.1.0/24  ← разрешить только локальную сеть
# git-daemon: ALL EXCEPT 192.168.1.0/24 : DENY
```

## Сравнение с другими серверами

```bash
# git daemon: простейший сервер
# + Встроен в Git, не требует установки
# + Быстрый протокол
# - Нет аутентификации
# - Порт 9418 часто заблокирован
# - Только для публичного чтения

# SSH: рекомендуется для приватных репозиториев
# + Аутентификация по ключам
# + Порт 22 обычно открыт
# + Шифрование

# HTTPS: рекомендуется для CI/CD
# + Порт 443 открыт везде
# + Аутентификация токенами
# + TLS шифрование
```

## Отладка

```bash
# Запуск с подробным выводом
git daemon --verbose --base-path=/srv/git --export-all /srv/git

# Тест клонирования
GIT_TRACE=1 git clone git://localhost/project.git

# Проверить логи systemd
journalctl -u git-daemon -f

# Проверить права на файлы
ls -la /srv/git/
# Пользователь git должен иметь права на чтение репозиториев
```

## Часто задаваемые вопросы

**Почему git clone git:// завершается с ошибкой?** Проверьте: 1) запущен ли git daemon, 2) открыт ли порт 9418, 3) существует ли репозиторий по указанному пути, 4) есть ли файл `git-daemon-export-ok` если не используется `--export-all`.

**Можно ли использовать git daemon в продакшене?** Для публичных репозиториев — да, но лучше использовать HTTPS через nginx с `git http-backend`. Для приватных — только SSH или HTTPS с аутентификацией.

**Как разрешить push только для определённых IP?** git daemon не имеет встроенного ACL. Используйте firewall (iptables/nftables) для ограничения доступа по IP к порту 9418.

**Какой максимальный размер репозитория для git daemon?** Нет технических ограничений. Ограничения определяются пропускной способностью сети и объёмом диска.

**Чем git daemon отличается от Smart HTTP?** git daemon использует собственный протокол git:// на порту 9418. Smart HTTP использует HTTPS на порту 443 и требует веб-сервера. Smart HTTP поддерживает аутентификацию и работает через прокси.

## Заключение

`git daemon` — простейший способ раздавать Git репозитории в локальной сети. Запускается одной командой, без конфигурации. Подходит для публичных репозиториев и внутренних сетей. Для серьёзных задач используйте [собственный Git сервер]({{< relref "sobstvennyj-git-server" >}}) на основе Gitea или GitLab CE с HTTPS и аутентификацией.
