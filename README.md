# PHP-Nginx-TCP — учебный стек на Docker (современная замена XAMPP/MAMP/Open Server)

Простая, воспроизводимая и «говорящая» среда для изучения PHP и его экосистемы. Стек собирается из контейнеров Docker и предназначен для локальных экспериментов.

Важное: этот проект предназначен исключительно для обучения, практики и ознакомления. Не используйте его в проде.

## Что внутри (архитектура)

Сервисы docker-compose.yml:
- PHP-FPM 8.4 (контейнер php-nginx-tcp) — выполняет PHP, слушает TCP:9000, Xdebug установлен, управляется переменными окружения.
- Nginx (контейнер nginx-tcp) — отдаёт статику и проксирует .php в PHP-FPM по TCP (fastcgi_pass php-nginx-tcp:9000); доступен на http://localhost:80.
- PostgreSQL (контейнер postgres-nginx-tcp) — база данных на localhost:5432, данные в именованном томе postgres-data.
- pgAdmin 4 (контейнер pgadmin) — веб-интерфейс PostgreSQL на http://localhost:8080.

Здоровье (healthchecks):
- PHP-FPM — проверка fastcgi по TCP (cgi-fcgi -connect 127.0.0.1:9000).
- Nginx — HTTP-запрос к http://localhost/.
- PostgreSQL — pg_isready.
- pgAdmin — HTTP-запрос к http://localhost:8080/.

Порядок старта: nginx-tcp ожидает, когда php-nginx-tcp станет healthy.

## Структура репозитория (актуальная)
```

php-nginx-tcp/
├── Makefile
├── README.md
├── config/
│   ├── nginx/
│   │   └── nginx.conf          # Конфиг Nginx (проксирование в PHP-FPM по TCP:9000)
│   └── php/
│       └── php.ini             # Конфиг PHP (dev-настройки + Xdebug через env)
├── docker/
│   └── php.Dockerfile          # Образ PHP-FPM 8.4 (Alpine) + расширения + Xdebug + Composer
├── docker-compose.yml          # Основной стек: PHP-FPM (TCP:9000), Nginx (fastcgi), PostgreSQL, pgAdmin
├── docker-compose.xdebug.yml   # Оверлей для включения Xdebug (mode=start)
├── docs/
│   ├── AI-CONTEXT.md           # Контекст/гайдлайны для AI
│   └── enhancement-plan.md     # Идеи по улучшению
├── env/
│   └── .env.example            # Пример переменных окружения (скопируйте в env/.env)
└── public/                     # DocumentRoot (монтируется в Nginx и PHP-FPM)
├── index.html
├── index.php
└── phpinfo.php
```
Обратите внимание: папки src/ и logs/ отсутствуют. Для обучения достаточно размещать PHP-файлы в public/.

## Быстрый старт

Предпосылки:
- Docker 20.10+
- Docker Compose v2+

Шаги:
1) Клонируйте репозиторий и перейдите в каталог проекта.
2) Скопируйте пример env:
    - mkdir -p env && cp env/.env.example env/.env
    - при необходимости отредактируйте пароли/имена БД.
3) Запустите стек:
    - make up (или docker compose up -d)
4) Проверьте доступность:
    - Web: http://localhost
    - pgAdmin: http://localhost:8080 (сервер postgres-nginx-tcp)
    - PostgreSQL: localhost:5432

Полезные команды Makefile:
- make setup — создать env/.env из примера
- make up / make down / make restart — управление стеком
- make logs / make status — логи и статусы контейнеров
- make xdebug-up / make xdebug-down — запуск/остановка стека с включённым Xdebug

## Конфигурация

PHP (config/php/php.ini):
- error_reporting=E_ALL, display_errors=On — удобно учиться на ошибках
- memory_limit=256M, upload_max_filesize=20M, post_max_size=20M
- opcache включён, validate_timestamps=1 (код обновляется сразу)
- Xdebug управляется через переменные окружения (см. ниже)

Nginx (config/nginx/nginx.conf):
- Отдаёт статику из /var/www/html.
- Проксирует .php в PHP-FPM по TCP: fastcgi_pass php-nginx-tcp:9000.
- index включает index.php; корректные fastcgi_param для SCRIPT_FILENAME и DOCUMENT_ROOT.
- client_max_body_size согласуйте с upload_max_filesize/post_max_size.

Docker-образ PHP (docker/php.Dockerfile):
- База: php:8.4-fpm-alpine.
- Установлены расширения: pdo, pdo_pgsql, pgsql, mbstring, xml, gd, bcmath, zip.
- Установлен Xdebug (через pecl), Composer, fcgi (для healthcheck).
- PHP-FPM слушает TCP:9000; порт не публикуется наружу, используется только внутри сети Docker.

Работа по TCP:9000 между Nginx и PHP-FPM:
- Общие тома для сокета не нужны; связь по сети в одной docker-сети.

## Переменные окружения (env/.env)

Минимальный набор (см. env/.env.example):
- POSTGRES_USER — имя пользователя PostgreSQL
- POSTGRES_PASSWORD — пароль пользователя
- POSTGRES_DB — имя создаваемой БД
- PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD — учётка для входа в pgAdmin
- Переменные Xdebug (см. ниже)

## Xdebug: как включить

По умолчанию Xdebug установлен, но выключен (переменные не заданы). Включить можно двумя способами:

Вариант A: оверлейный compose-файл
- make xdebug-up
  (эквивалент docker compose -f docker-compose.yml -f docker-compose.xdebug.yml up -d)
- Внутри php.ini используются переменные XDEBUG_MODE=debug и XDEBUG_START=yes.

Вариант B: задать переменные в env/.env и перезапустить php-контейнер
- XDEBUG_MODE=debug
- XDEBUG_START=yes
- затем docker compose up -d --no-deps php-nginx-tcp

IDE: подключение по Xdebug 3 на порт 9003, client_host=host.docker.internal.

## Рабочие директории и монтирование

- public/ монтируется в /var/www/html одновременно в PHP-FPM и Nginx — любые изменения видны сразу.
- config/php/php.ini монтируется в /usr/local/etc/php/conf.d/local.ini (только чтение).
- Для PostgreSQL используется именованный том postgres-data (персистентные данные).

## Подключение к PostgreSQL из PHP (пример)
```
php
<?php
$host = 'postgres-nginx-tcp';
$port = 5432;
$dbname = 'your-db-name';
$user = 'your-user';
$pass = 'your-user-password';

$dsn = "pgsql:host=$host;port=$port;dbname=$dbname";
$pdo = new PDO($dsn, $user, $pass, [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
]);
```
## Решение проблем

502/404 на .php:
- Проверьте fastcgi_param SCRIPT_FILENAME ($document_root$fastcgi_script_name) и совпадение путей в Nginx и PHP-FPM.

Ограничения загрузки файлов:
- client_max_body_size (Nginx) должен быть согласован с upload_max_filesize и post_max_size (PHP).

Порты заняты:
- Измените привязку в docker-compose.yml, например 8081:80 для Nginx, 5433:5432 для PostgreSQL.

Контейнеры не стартуют по порядку:
- Проверьте healthchecks командой docker compose ps; nginx-tcp зависит от healthy php-nginx-tcp.

Xdebug не подключается:
- Проверьте, что используете порт 9003 в IDE, и что XDEBUG_MODE/START заданы (compose.xdebug.yml или env/.env).

## Дисклеймер

Проект создан для обучения и экспериментов с PHP-стеком. Не предназначен для production-использования или оценки производительности.

—
Если нужна шпаргалка по архитектуре и договорённостям для AI, см. docs/AI-CONTEXT.md.
```
