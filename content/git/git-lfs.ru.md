---
title: "Git LFS: как хранить большие файлы (видео, модели, архивы) в Git"
description: "Git LFS (Large File Storage) хранит большие файлы отдельно от репозитория. Установка, настройка git lfs track, лимиты GitHub, миграция существующих файлов."
date: 2026-01-30
lastmod: 2026-01-30
draft: false
slug: "git-lfs"
keywords: ["git lfs", "git large file storage", "большие файлы в git", "git lfs установка", "git lfs track"]
tags: ["git", "intermediate"]
categories: ["git"]
---

Git был спроектирован для управления текстовыми файлами исходного кода, а не для больших двоичных файлов. Когда вы добавляете видео размером 500 МБ или модель машинного обучения в 1 ГБ в обычный Git репозиторий, происходит следующее:

1. Весь файл хранится в каждом коммите (даже если вы его едва изменили)
2. Каждый `git clone` загружает всю историю, включая все версии этого файла
3. Репозиторий раздувается до гигабайтов
4. Операции становятся медленными и неудобными

Git LFS (Large File Storage) решает эту проблему, храня большие файлы отдельно от репозитория. В самом Git хранятся только маленькие текстовые "указатели", указывающие на реальные файлы в отдельном хранилище. Это позволяет командам работать с большими файлами так же просто, как с текстовыми файлами, но без всех проблем.

## Как работает Git LFS

Давайте разберемся, что происходит под капотом.

**Без Git LFS:**
```
Repository on GitHub
│
├── README.md (1 КБ)
├── src/
│   └── main.py (10 КБ)
└── models/
    └── model.pkl (500 МБ) ← Весь файл хранится в Git
        └── История: версия 1, версия 2, версия 3 (1.5 ГБ всего)
```

При клонировании вам загружаются все 1.5 ГБ.

**С Git LFS:**
```
Repository on GitHub
│
├── README.md (1 КБ)
├── src/
│   └── main.py (10 КБ)
└── models/
    └── model.pkl (pointer файл, 130 байт)
        └── версия 1 → хранится в LFS storage (500 МБ)
        └── версия 2 → хранится в LFS storage (520 МБ)
        └── версия 3 → хранится в LFS storage (480 МБ)
```

Содержимое `model.pkl` выглядит так:
```
version https://git-lfs.github.com/spec/v1
oid sha256:2c26b46911185131006504cda96a0901e61c6bdb8fe9adc881d0f1f82410a42d
size 500000000
```

При клонировании вам загружается только pointer (130 байт), а сам файл загружается только когда вам нужен. Это может быть ускорено параллельной загрузкой.

## Установка Git LFS

### На macOS

```bash
# Через Homebrew
brew install git-lfs

# Инициализация
git lfs install
```

### На Ubuntu/Debian

```bash
# Метод 1: через пакет
curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash
sudo apt-get install git-lfs

# Метод 2: прямая загрузка
wget https://github.com/git-lfs/git-lfs/releases/download/v3.5.0/git-lfs-linux-amd64-v3.5.0.tar.gz
tar xzf git-lfs-linux-amd64-v3.5.0.tar.gz
sudo ./git-lfs-linux-amd64-v3.5.0/install.sh
```

### На Windows

```bash
# Через Chocolatey
choco install git-lfs

# Или скачайте установщик с GitHub
# https://github.com/git-lfs/git-lfs/releases

# Инициализация
git lfs install
```

### Проверка установки

```bash
git lfs version
# git-lfs/3.5.0 (GitHub; windows amd64; go 1.21.5)

git lfs install
# Git LFS initialized.
```

## Настройка отслеживания файлов

После установки Git LFS вам нужно указать, какие файлы должны храниться в LFS. Используйте `git lfs track` для добавления расширений файлов или паттернов.

### Отслеживание одного типа файлов

```bash
# Отслеживать все .psd файлы (Photoshop)
git lfs track "*.psd"

# Отслеживать все видео файлы
git lfs track "*.mp4"
git lfs track "*.mov"
git lfs track "*.avi"

# Отслеживать модели машинного обучения
git lfs track "*.pkl"
git lfs track "*.h5"
git lfs track "*.pt"
```

### Отслеживание по директориям

```bash
# Отслеживать все файлы в папке
git lfs track "assets/*"
git lfs track "models/*.pkl"
```

### Просмотр отслеживаемых файлов

Когда вы выполняете `git lfs track`, Git автоматически создает или обновляет файл `.gitattributes`:

```bash
# Посмотрите содержимое .gitattributes
cat .gitattributes
```

Содержимое будет выглядеть так:
```
*.psd filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.pkl filter=lfs diff=lfs merge=lfs -text
*.h5 filter=lfs diff=lfs merge=lfs -text
```

**Важно:** добавьте `.gitattributes` в Git, так как это нужно для правильной работы LFS на другие машинах:

```bash
git add .gitattributes
git commit -m "Add LFS tracking configuration"
```

## Добавление больших файлов

После настройки отслеживания просто добавляйте и коммитьте файлы как обычно:

```bash
# Скопируйте большие файлы в репозиторий
cp ~/Downloads/model.pkl models/
cp ~/Downloads/video.mp4 assets/

# Добавьте их в Git
git add models/model.pkl assets/video.mp4

# Или добавьте всё отслеживаемые файлы
git add .

# Коммитьте как обычно
git commit -m "Add trained model and demo video"

# Push как обычно
git push origin main
```

Git LFS автоматически заменит файлы на pointers перед отправкой, а сами файлы пойдут в LFS storage.

## Миграция существующих больших файлов

Если вы уже добавили большие файлы в Git обычным способом, вы можете мигрировать их на LFS с помощью `git-filter-repo`:

### Установка git-filter-repo

```bash
# Через pip (требуется Python 3)
pip install git-filter-repo

# Или на macOS
brew install git-filter-repo
```

### Миграция файлов

```bash
# Проверьте большие файлы в истории
git lfs migrate info --everything --above=10M

# Мигрируйте файлы размером > 10 МБ
git lfs migrate import --everything --above=10M

# Или мигрируйте конкретный тип файлов
git lfs migrate import --everything --include="*.pkl"

# Мигрируйте большие файлы до определенного коммита
git lfs migrate import --no-rewrite --include="*.mp4" main
```

После миграции вам потребуется force push:

```bash
# Backup на случай ошибки
git branch backup

# Force push (будьте осторожны!)
git push origin --force --all
git push origin --force --tags
```

## Просмотр LFS файлов в репозитории

```bash
# Посмотрите все отслеживаемые LFS файлы
git lfs ls-files

# Более подробная информация
git lfs ls-files -l
# a234567890bcdef1234567890bcdef1234567890 * models/model.pkl (500.0 MB)
# b345678901cdef1234567890bcdef1234567890 * assets/video.mp4 (1.2 GB)

# Статус LFS файлов
git lfs status
# On branch main
# LFS objects to be committed:
#   models/model.pkl (new file)
# LFS objects not staged for commit:

# Посмотрите содержимое pointer файла
cat models/model.pkl
# version https://git-lfs.github.com/spec/v1
# oid sha256:2c26b46911185131006504cda96a0901e61c6bdb8fe9adc881d0f1f82410a42d
# size 500000000
```

## Лимиты и цены

### GitHub

- **Бесплатно:** 1 ГБ в месяц пропускной способности загрузки + 1 ГБ хранилища
- **Платные пакеты:** $5 за пакет 50 ГБ пропускной способности + 50 ГБ хранилища

Для активных проектов с большими файлами это может быть дороговато.

### GitLab

- **Бесплатно:** 5 ГБ хранилища LFS на группу (на свежем плане)
- **Premium:** неограниченное хранилище

### Self-hosted Git (Gitea, GitLab Community)

- **Неограниченное** LFS хранилище на вашем сервере
- Вам просто нужно место на диске сервера

### AWS S3, MinIO и другие хранилища

Многие платформы позволяют конфигурировать внешнее LFS хранилище на S3, MinIO или других сервисах.

## Клонирование без загрузки LFS файлов

Иногда вам нужна структура репозитория, но не сами большие файлы (например, для CI/CD скриптов). Используйте этот трюк:

```bash
# Клонируйте без LFS файлов (только pointers)
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/user/repo.git

# Потом, если понадобиться, загрузите нужные файлы
git lfs pull --include="*.pkl"

# Или загрузите всё
git lfs pull
```

Это экономит время и пропускную способность, если вам не нужны все файлы.

## Интеграция с CI/CD

В GitHub Actions убедитесь, что LFS файлы загружаются в workflow:

```yaml
name: Build
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          lfs: true  # Загрузить LFS файлы

      - name: Use model
        run: python train.py models/model.pkl
```

Для других CI/CD систем убедитесь, что:
1. Git LFS установлен в рабочей среде
2. Переменные окружения содержат credentials для доступа к LFS
3. `git lfs pull` выполняется после clone

## FAQ

**В: Разница между .gitignore и git lfs track?**
О: `.gitignore` говорит Git не отслеживать файлы вообще. `git lfs track` говорит Git отслеживать файлы, но хранить их в LFS вместо основного репозитория. Используйте gitignore для вспомогательных файлов (build artifacts), и LFS для нужных версионировать больших файлов.

**В: Мой хостинг не поддерживает Git LFS. Что делать?**
О: Используйте:
- Самостоятельное хранилище LFS через git-lfs-server
- S3-compatible storage (MinIO, DigitalOcean Spaces)
- Gitea с встроенной поддержкой LFS
- Другие платформы (GitLab, Forgejo)

**В: Могу ли я клонировать репозиторий с LFS на машину без Git LFS?**
О: Вы сможете клонировать и получите pointer файлы (текстовые), но не сами большие файлы. Git LFS нужен, чтобы получить реальные файлы.

**В: Как удалить большой файл из истории, если я случайно добавил его без LFS?**
О: Используйте `git-filter-repo` (как показано выше в миграции) или `git filter-branch`. Это потребует force push, что повлияет на всех, работающих с репозиторием.

**В: Как это влияет на производительность?**
О: LFS добавляет минимальную задержку (дополнительный запрос для скачивания файла), но это компенсируется намного более быстрым клонированием и операциями с историей. Для больших файлов это огромное улучшение.

## Практические примеры

### Пример 1: Machine Learning проект

```bash
# Инициализация LFS
git lfs install

# Отслеживание моделей и датасетов
git lfs track "*.pkl"
git lfs track "*.h5"
git lfs track "*.joblib"
git lfs track "data/*.csv"

# Добавить .gitattributes
git add .gitattributes

# Работать как обычно
cp trained_model.pkl models/
git add models/trained_model.pkl
git commit -m "Add trained model v2"
git push
```

### Пример 2: Game Development

```bash
# Отслеживать все ассеты
git lfs track "Assets/**"
git lfs track "*.unitypackage"
git lfs track "*.blend"
git lfs track "*.fbx"

# .gitattributes будет содержать
cat .gitattributes
# Assets/** filter=lfs diff=lfs merge=lfs -text
# *.unitypackage filter=lfs diff=lfs merge=lfs -text
```

### Пример 3: Video Content

```bash
git lfs track "*.mp4"
git lfs track "*.mov"
git lfs track "*.mkv"

# Добавить видео
git add videos/demo.mp4
git commit -m "Add demo video"
git push
```

## Заключение

Git LFS — это стандартное решение для управления большими файлами в Git. Оно особенно полезно для:

- **Game Development** (ассеты, текстуры, звуки)
- **Machine Learning** (модели, датасеты)
- **Video/Media Production** (видео, аудио файлы)
- **CAD и Design** (PSD, DWG, 3D модели)
- **Архивы и backup** (большие архивы)

Ключевые моменты:
1. Установите git-lfs один раз
2. Используйте `git lfs track` для типов файлов
3. Коммитьте и пушьте как обычно
4. Git LFS заботится об остальном

Если вы работаете с большими файлами без LFS, то вы просто усложняете жизнь себе и своей команде. Потратьте 5 минут на настройку — и работа станет намного приятнее.

Читайте также:
- {{< relref "gitignore" >}} — как правильно использовать .gitignore
- {{< relref "git-push" >}} — детали команды push
- {{< relref "git-filter-branch" >}} — переписывание истории
