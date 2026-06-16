---
title: Установка
layout: default
nav_order: 1
parent: Oly-exams
---

# Руководство по установке портала oly-exams
{: .no_toc }

<details open markdown="block" class="toc-h2-only">
  <summary>Содержание</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

- **Репозиторий:** <https://gitlab.com/oly-exams/exam_tools>
- **Документация:** <https://oly-exams.org/docs/>
- **Официальный сайт:** <https://www.oly-exams.org/>
- **Исходное руководство:** [https://docs.mamrenko.ru](https://docs.mamrenko.ru/share/tnkf9x3rqb/p/rukovodstvo-po-ustanovke-i-nastrojke-portala-oly-exams-gGFXbWdxJ3)

---

## Системные требования

| Параметр | Требование |
|----------|------------|
| ОС | Ubuntu 24.04 LTS (рекомендуется) или другие Linux дистрибутивы |
| Python | 3.8+ (рекомендуется 3.10+) |
| Память | минимум 2 GB RAM |
| Диск | минимум 10 GB свободного места |
| Сеть | доступ к интернету для установки зависимостей |

---

## Предварительная подготовка

### 1. Установка системных зависимостей

```bash
# Обновляем систему
sudo apt update && sudo apt upgrade -y

# Устанавливаем необходимые пакеты
sudo apt install -y curl build-essential git python3-dev libpq-dev sqlite3 nginx

# Устанавливаем Node.js и npm для bower
sudo apt install -y nodejs npm
sudo npm install -g bower
```

### 2. Установка uv (современный менеджер Python пакетов; можно использовать pip или poetry)

```bash
# Установка uv через официальный скрипт
curl -LsSf https://astral.sh/uv/install.sh | sh

# Добавляем uv в PATH
source $HOME/.local/bin/env
```

---

## Основная установка

### 1. Создание директории проекта и клонирование

```bash
# Клонируем репозиторий
git clone https://gitlab.com/oly-exams/exam_tools
cd exam_tools/
```

### 2. Установка Python зависимостей

```bash
# Синхронизируем зависимости (создает виртуальное окружение автоматически)
uv sync

# На июнь 2026 не все зависимости корректно отрабатывают на актуальных версиях Python
# Например, не устанавливается pillow==9.5.0 для Python 3.11+
# Решается uv add pillow или uv add pillow==10.1.0
# https://pillow.readthedocs.io/en/latest/installation/python-support.html

# Проверяем установку Django
uv run python manage.py check
```

### 3. Настройка конфигурации

```bash
# Копируем пример настроек
cp exam_tools/settings_example.py exam_tools/settings.py

# Редактируем настройки
nano exam_tools/settings.py
```

Основные параметры для настройки в `settings.py`:

```python
# Базовые настройки
DEBUG = False  # Для продакшена
ALLOWED_HOSTS = ['your-domain.com', 'www.your-domain.com']

# База данных (по умолчанию SQLite)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': 'ipho.db',
    }
}

# Настройки безопасности для продакшена
SECRET_KEY = 'your-secret-key-here'  # Сгенерировать новый
SECURE_SSL_REDIRECT = True  # Если используете HTTPS
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
```

### 4. Установка Frontend зависимостей

```bash
# Переходим в директорию bower и устанавливаем зависимости
cd bower/
bower install
cd ..
```

### 5. Инициализация базы данных

```bash
# Применяем миграции
uv run python manage.py migrate

# Собираем статические файлы (соберутся в папку PROJECT_PATH / collected_static)
uv run python manage.py collectstatic --noinput

# Создаем суперпользователя (для доступа в админку)
uv run python manage.py createsuperuser
```

### 6. Загрузка тестовых данных (опционально)

```bash
# Переходим в директорию с данными
cd ipho_data/

# Загружаем mock данные для тестирования
uv run python postgres_testing_initial_data.py --dataset mock

# Или загружаем данные конкретной олимпиады
uv run python postgres_testing_initial_data.py --dataset ibo2019
```

---

## Настройка веб-сервера (Nginx + Gunicorn)

### 1. Установка Gunicorn

```bash
# Устанавливаем Gunicorn через uv
uv add gunicorn
```

### 2. Создание конфигурации Nginx

```bash
# Создаем конфигурацию для сайта
sudo tee /etc/nginx/sites-available/your-domain.conf >/dev/null <<'NGINX'
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com www.your-domain.com;

    # Увеличиваем лимит загрузки файлов
    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_read_timeout 300;
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
    }

    # Статические файлы (рекомендуется для продакшена)
    location /static/ {
        alias /path/to/your/project/exam_tools/collected_static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /path/to/your/project/exam_tools/media/;
        expires 1y;
        add_header Cache-Control "public";
    }
}
NGINX

# Активируем сайт
sudo ln -s /etc/nginx/sites-available/your-domain.conf /etc/nginx/sites-enabled/

# Проверяем конфигурацию
sudo nginx -t

# Перезагружаем Nginx
sudo systemctl reload nginx
```

```bash
# Может потребоваться удалить дефолтный конфиг nginx
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

### 3. Настройка прав доступа для статических файлов

```bash
# Даем права на чтение статических файлов для Nginx
sudo chmod o+rx /home
sudo chmod o+rx /home/username
sudo chmod o+rx /home/username/your-domain.com
sudo chmod o+rx /home/username/your-domain.com/exam_tools

# Устанавливаем права для статических файлов
sudo find /path/to/your/project/exam_tools/collected_static -type d -exec chmod 755 {} \;
sudo find /path/to/your/project/exam_tools/collected_static -type f -exec chmod 644 {} \;
```

### 4. Запуск приложения через Gunicorn

```bash
# Тестовый запуск
uv run gunicorn exam_tools.wsgi:application --bind 127.0.0.1:8000 --workers 3
```

```bash
# Для постоянного запуска используйте screen
screen -S oly_exams_server
uv run gunicorn exam_tools.wsgi:application --bind 127.0.0.1:8000 --workers 3

# Нажмите Ctrl+A, затем D для отсоединения от screen
```

```bash
# или настройте gunicorn как системный сервис
sudo nano /etc/systemd/system/gunicorn.service
```

```ini
[Unit]
Description=gunicorn daemon for exam_tools
After=network.target

[Service]
User=root
WorkingDirectory=/path/to/your/project/exam_tools
ExecStart=/path/to/your/uv run gunicorn exam_tools.wsgi:application --bind 127.0.0.1:8000 --workers 3
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# запуск сервиса
sudo systemctl daemon-reload
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
```

---

## Настройка HTTPS с Let's Encrypt (обязательно)

Если не хотите HTTPS, в `exam_tools/settings.py` нужно установить параметр `SECURE_SSL_REDIRECT = False`.

```bash
# Устанавливаем Certbot
sudo apt install -y certbot python3-certbot-nginx

# Получаем SSL сертификат
sudo certbot --nginx -d your-domain.com -d www.your-domain.com

# Проверяем автообновление сертификатов
sudo certbot renew --dry-run
```

---

## Настройка компиляции LaTeX (XeLaTeX + RabbitMQ + Celery)

Портал компилирует экзаменационные материалы в PDF с помощью XeLaTeX. Компиляция выполняется асинхронно через очередь задач: RabbitMQ принимает задания, Celery-воркер их обрабатывает.

### 1. Установка XeLaTeX и необходимых пакетов

```bash
sudo apt install -y texlive texlive-xetex texlive-lang-cjk texlive-lang-greek \
  fonts-roboto fonts-liberation \
  python3-cairosvg pandoc
```

Пакет `ttf-mscorefonts-installer` (шрифты Microsoft) требует репозитория multiverse:

```bash
sudo add-apt-repository multiverse
sudo apt update
sudo apt install -y ttf-mscorefonts-installer
```

Проверка:

```bash
xelatex --version
```

### 2. Установка и запуск RabbitMQ

```bash
sudo apt install -y rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
```

Проверка:

```bash
sudo rabbitmq-diagnostics status
```

RabbitMQ должен слушать порт `5672`. Конфигурационный файл проекта находится в `docker/secrets/unsafe/rabbitmq.conf` и использует стандартные учётные данные `guest/guest` — они совпадают с дефолтными настройками RabbitMQ, поэтому дополнительная настройка не требуется.

### 3. Запуск Celery-воркера как systemd-сервиса

Создайте файл сервиса:

```bash
sudo tee /etc/systemd/system/celery.service > /dev/null << 'EOF'
[Unit]
Description=Celery worker for exam_tools
After=network.target rabbitmq-server.service
Requires=rabbitmq-server.service

[Service]
User=root
WorkingDirectory=/path/to/your/project/exam_tools
Environment=C_FORCE_ROOT=1
ExecStart=/path/to/your/uv run celery -A exam_tools worker -E --concurrency=1
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

> **Примечание:** `C_FORCE_ROOT=1` необходим если сервис запускается от пользователя root. Для продакшена рекомендуется создать отдельного пользователя и убрать эту переменную.

Активируйте и запустите:

```bash
sudo systemctl daemon-reload
sudo systemctl enable celery
sudo systemctl start celery
sudo systemctl status celery
```

В выводе `status` должна присутствовать строка:

```
transport: amqp://guest:**@localhost:5672//
```

### 4. Проверка работы

```bash
# Статус воркера
sudo systemctl status celery

# Логи в реальном времени
journalctl -u celery -f
```

### Итоговые сервисы

| Сервис | Команда управления |
|--------|-------------------|
| RabbitMQ | `sudo systemctl [start\|stop\|restart] rabbitmq-server` |
| Celery | `sudo systemctl [start\|stop\|restart] celery` |
| Gunicorn | `sudo systemctl [start\|stop\|restart] gunicorn` |

Все три сервиса настроены на автозапуск при старте сервера.

---

## Обслуживание и администрирование

### Полезные команды для управления

```bash
# Проверка статуса приложения
uv run python manage.py check

# Применение новых миграций
uv run python manage.py migrate

# Обновление статических файлов
uv run python manage.py collectstatic --noinput

# Создание резервной копии базы данных (SQLite)
cp ipho.db ipho_backup_$(date +%Y%m%d_%H%M%S).db

# Просмотр логов Nginx
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# Управление screen сессиями
screen -ls                    # Список сессий
screen -r session_name        # Подключение к сессии
screen -S new_session_name    # Создание новой сессии
```

### Обновление проекта

```bash
# Останавливаем приложение
sudo systemctl stop oly-exams

# Обновляем код
git pull origin master

# Обновляем зависимости
uv sync

# Применяем миграции
uv run python manage.py migrate

# Собираем статические файлы
uv run python manage.py collectstatic --noinput

# Запускаем приложение
sudo systemctl start oly-exams
```

---

## Настройка документации (опционально)

```bash
# Устанавливаем зависимости для документации
uv add mkdocs mkdocs-material pymdown-extensions

# Собираем документацию
uv run mkdocs build

# Проверьте, что в папке exam_tools появилась папка _docs_html
```

Добавьте в конфигурацию Nginx:

```nginx
location /docs {
    alias /path/to/your/project/exam_tools/_docs_html/;
    index index.html;
    try_files $uri $uri/ $uri/index.html =404;
}
```

Перезапустить Nginx

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Могут возникнуть проблемы с правами, тогда нужно выдать Nginx права на папку `_docs_html`
```bash
# Дать Nginx доступ к папке
chmod 755 /path/to/your/project

# Дать доступ к папке с документацией
chmod -R 755 /path/to/your/project/exam_tools/_docs_html/
```

---

## Решение распространённых проблем

### 1. Ошибки прав доступа к статическим файлам

```bash
# Проверьте права доступа
ls -la /path/to/static/files

# Исправьте права если необходимо
sudo chown -R www-data:www-data /path/to/static/files
sudo chmod -R 755 /path/to/static/files
```

### 2. Проблемы с базой данных

```bash
# Проверьте подключение к БД
uv run python manage.py dbshell

# Сброс и пересоздание БД (осторожно!)
rm ipho.db
uv run python manage.py migrate
uv run python manage.py createsuperuser
```

---

## Заключение

Для получения дополнительной помощи обращайтесь к официальной документации: https://oly-exams.org/docs/
