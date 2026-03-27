---
title: "Forgejo: установка, настройка и Git LFS"
description: "Что такое Forgejo, как установить и настроить. Настройка Git LFS в Forgejo. Сравнение с Gitea, GitLab и GitHub. Forgejo как самостоятельный git-хостинг."
date: 2024-12-18
lastmod: 2024-12-18
draft: false
slug: "forgejo"
keywords: ["forgejo", "forgejo установка", "forgejo git lfs настройка", "forgejo vs gitea", "forgejo vs gitlab", "forgejo самохостинг", "forgejo docker", "forgejo git lfs", "что такое forgejo", "forgejo настройка"]
tags: ["git", "hosting", "self-hosted"]
categories: ["git"]
aliases: []
---

**Forgejo** — это форк платформы Gitea, появившийся в 2022 году как ответ на изменения в управлении исходным проектом. По сути, это лёгкий, простой в развёртывании самостоятельный git-хостинг, написанный на Go, но с фокусом на сообщество и полную открытость.

### Почему появился Forgejo

Gitea долгое время была отличным решением для self-hosted git, но в какой-то момент началась коммерциализация проекта, и сообщество было обеспокоено будущим направлением развития. Поэтому создатели Forgejo решили сделать идеологический форк, который остался бы полностью открытым, community-driven и без коммерческого давления.

### Ключевые отличия

**От Gitea**: Forgejo полностью совместим с Gitea API и базой данных, но имеет собственное развитие и приоритеты. Вы даже можете мигрировать между ними без потери данных.

**От GitLab**: Forgejo намного легче и проще в настройке. GitLab требует больше ресурсов, имеет множество функций, которые не всем нужны. Forgejo — это выбор для тех, кому нужна простота.

**От GitHub**: GitHub — облачный сервис, Forgejo развёртывается на вашем сервере. У вас полный контроль над данными.

---

## Когда выбирать Forgejo

Forgejo идеален в следующих сценариях:

- **Для команд** — нужен self-hosted git без тяжелого GitLab, но с надежностью
- **Для организаций** — требования к приватности и полный контроль над данными
- **Для личного проекта** — создать собственный git-сервер на своём сервере или VPS
- **Для обучения** — учебная платформа для работы с git в корпоративной среде
- **Для интеграции** — API совместим с Gitea, хорошо работает с CI/CD

---

## Установка через Docker (самый простой способ)

Самый удобный путь установки Forgejo — использование Docker и Docker Compose. Вам нужны только два инструмента и два файла.

### Шаг 1: Создайте docker-compose.yml

```yaml
version: "3"
services:
  forgejo:
    image: codeberg.org/forgejo/forgejo:9
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    volumes:
      - ./forgejo-data:/data
      - /etc/timezone:/etc/timezone:ro
    ports:
      - "3000:3000"
      - "222:22"
```

### Шаг 2: Запустите контейнер

```bash
docker compose up -d
```

### Шаг 3: Первоначальная настройка

Откройте браузер и перейдите на `http://localhost:3000`. Вы увидите мастер установки, где нужно:

1. Установить тип базы данных (SQLite по умолчанию — отлично для небольших установок)
2. Задать параметры администратора
3. Настроить SMTP для отправки писем (опционально)

Всё просто и интуитивно.

---

## Настройка app.ini

После первоначальной установки вам потребуется отредактировать файл конфигурации `app.ini`. Он находится в томе `/data/forgejo/conf/app.ini` внутри контейнера или в `./forgejo-data/forgejo/conf/app.ini` на хосте.

### Ключевые параметры

**Серверные настройки:**

```ini
[server]
DOMAIN = git.example.com
ROOT_URL = https://git.example.com/
SSH_DOMAIN = git.example.com
OFFLINE_MODE = false
PROTOCOL = https
CERT_FILE = /etc/ssl/certs/cert.pem
KEY_FILE = /etc/ssl/private/key.pem
```

**Параметры базы данных:**

```ini
[database]
DB_TYPE = sqlite3
PATH = /data/forgejo/forgejo.db
```

**Email (SMTP):**

```ini
[mailer]
ENABLED = true
SMTP_ADDR = mail.example.com
SMTP_PORT = 587
FROM = forgejo@example.com
USER = your-email@example.com
PASSWD = your-password
```

После редактирования перезагрузите контейнер:

```bash
docker compose restart forgejo
```

---

## Настройка Git LFS (главное)

Git LFS (Large File Storage) — это расширение Git для хранения больших файлов. Это критично, если вы работаете с видео, изображениями, бинарными файлами или моделями машинного обучения.

### Включение LFS в Forgejo

В файле `app.ini` найдите или создайте секцию `[server]` и добавьте:

```ini
[server]
LFS_START_SERVER = true
LFS_JWT_SECRET = your-random-secret-key-here-32-chars-minimum
```

Замените `your-random-secret-key-here-32-chars-minimum` на случайную строку (минимум 32 символа). Вы можете сгенерировать её так:

```bash
openssl rand -base64 32
```

### Настройка хранилища LFS

По умолчанию LFS хранится на диске. Добавьте в `app.ini`:

```ini
[lfs]
PATH = /data/forgejo/lfs
MAX_FILE_SIZE = 0
```

`PATH` — это где хранятся LFS объекты. `MAX_FILE_SIZE = 0` означает без ограничений по размеру файла. Если хотите ограничить, например, 5 GB:

```ini
[lfs]
MAX_FILE_SIZE = 5368709120
```

### LFS в Docker

Добавьте том для LFS данных в `docker-compose.yml`:

```yaml
version: "3"
services:
  forgejo:
    image: codeberg.org/forgejo/forgejo:9
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    volumes:
      - ./forgejo-data:/data
      - ./forgejo-lfs:/data/forgejo/lfs
    ports:
      - "3000:3000"
      - "222:22"
```

После изменения:

```bash
docker compose up -d
```

### Использование Git LFS

Как только LFS включен, используйте его так:

**1. Инициализируйте LFS в репозитории:**

```bash
git lfs install
```

**2. Отслеживайте большие файлы:**

```bash
git lfs track "*.psd"
git lfs track "*.mp4"
git lfs track "*.zip"
git add .gitattributes
git commit -m "Add LFS tracking"
```

**3. Делайте обычный push:**

```bash
git add large-file.psd
git commit -m "Add design file"
git push
```

Forgejo автоматически обнаружит отслеживаемые файлы и будет хранить их в LFS.

### S3-совместимое хранилище

Если хотите хранить LFS файлы в S3 (Minio, AWS S3 и т.д.):

```ini
[lfs]
STORAGE_TYPE = minio
MINIO_ENDPOINT = s3.example.com:9000
MINIO_ACCESS_KEY_ID = your-access-key
MINIO_SECRET_ACCESS_KEY = your-secret-key
MINIO_BUCKET = forgejo-lfs
MINIO_USE_SSL = true
```

Это особенно полезно, если ваш сервер имеет ограниченное дисковое пространство.

---

## Forgejo Actions: встроенный CI/CD

Начиная с версии 3.x, Forgejo имеет встроенную систему CI/CD, совместимую с синтаксисом GitHub Actions.

### Включение Actions

В `app.ini`:

```ini
[actions]
ENABLED = true
```

### Первый workflow

Создайте файл `.forgejo/workflows/test.yml` в репозитории:

```yaml
name: Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: |
          echo "Running tests..."
          # Ваши команды тестирования
```

### Регистрация runner

Для выполнения workflows нужен runner. Скачайте его с сайта Forgejo и зарегистрируйте:

```bash
forgejo-runner register
```

Это простой процесс интерактивной регистрации, где вы укажете URL вашего Forgejo и токен.

---

## Миграция с GitHub/GitLab

Forgejo имеет встроенный мастер миграции. Просто при создании нового репозитория выберите "Migrate repository" и укажите URL исходного репозитория. Все issues, pull requests и wiki будут перенесены.

---

## Сравнение: Forgejo vs Gitea vs GitLab vs GitHub

| Параметр | Forgejo | Gitea | GitLab | GitHub |
|----------|---------|-------|--------|--------|
| **Лицензия** | MIT (open-source) | MIT (open-source) | Proprietary (Community Edition) | Proprietary |
| **Самохостинг** | Да | Да | Да (Community) | Нет |
| **Требуемые ресурсы** | 256 MB RAM | 256 MB RAM | 4+ GB RAM | — |
| **Git LFS** | Встроена | Встроена | Встроена | Встроена |
| **CI/CD** | Forgejo Actions | —* | GitLab CI | GitHub Actions |
| **API совместимость** | Gitea API | Forgejo API | GitLab API | GitHub API |
| **Community-driven** | Да (идеология) | Условно | Нет | Нет |

*Gitea не имеет встроенной CI/CD в базовой версии.

---

## FAQ

### Совместим ли Forgejo с Gitea?

Да, полностью. API совместим, база данных совместима. Вы можете экспортировать данные из Gitea и импортировать в Forgejo без потерь.

### Можно ли мигрировать с Gitea на Forgejo?

Абсолютно. Экспортируйте данные из Gitea, установите Forgejo и восстановите данные. Это практически не требует настройки благодаря совместимости.

### Какой порт использует Forgejo по умолчанию?

3000 (HTTP) и 22 (SSH для Git). В Docker мы маппируем SSH на порт 222, чтобы не конфликтовать с системным SSH.

### Как сделать бэкап?

Простой способ — скопировать том Docker:

```bash
docker exec forgejo tar czf /backup/forgejo-backup.tar.gz /data/
docker cp forgejo:/backup/forgejo-backup.tar.gz ./
```

Или просто скопируйте директорию `./forgejo-data` на внешний диск.

### Нужен ли Nginx/Apache перед Forgejo?

Рекомендуется для HTTPS и балансировки нагрузки, но для небольших установок Forgejo может работать и напрямую с HTTPS.

### Сколько пользователей может поддерживать одна установка?

Forgejo легко масштабируется на сотни пользователей с минимальными ресурсами. На отдельном сервере (2 CPU, 2 GB RAM) может работать 100+ активных пользователей.

---

## Заключение

**Forgejo** — это отличный выбор, если вы ищете простой, надежный и полностью открытый self-hosted git-хостинг. Установка через Docker занимает буквально 5 минут, а Git LFS настраивается в несколько строк конфигурации. Это идеально для команд, которым нужна приватность данных и полный контроль над своим git-сервером.

Начните с базовой установки Docker, настройте LFS для ваших больших файлов, и вы получите мощный инструмент для управления кодом без зависимости от облачных сервисов.

Больше информации вы найдёте на официальном сайте [forgejo.org](https://forgejo.org) и репозитории на [codeberg.org](https://codeberg.org/forgejo/forgejo).

---

## Связанные материалы

- [Git LFS: подробное руководство](/git/git-lfs)
- [Собственный git-сервер: пошаговое руководство](/git/sobstvennyj-git-server)
- [Git init --bare: создание репозитория](/git/git-init-bare)
- [GitLab: обзор функций](/git/gitlab-obzor)
- [GitHub vs GitLab: сравнение платформ](/git/github-vs-gitlab)

## По теме

- [Собственный git-сервер]({{< relref "sobstvennyj-git-server" >}})
- [Хостинги Git]({{< relref "git-hosting" >}})
- [Git LFS]({{< relref "git-lfs" >}})
- [Обзор GitLab]({{< relref "gitlab-obzor" >}})
- [GitHub vs GitLab]({{< relref "github-vs-gitlab" >}})
