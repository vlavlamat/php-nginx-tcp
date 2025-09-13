AI-CONTEXT — руководящие принципы для AI Assistant и Junie (проект PHP-FPM + Nginx по TCP:9000 + PostgreSQL + pgAdmin)

1) Назначение документа
- Зафиксировать цели и рамки проекта.
- Синхронизировать подходы AI Assistant и Junie, чтобы не сбиваться с курса.
- Служить источником правды при принятии технических решений в стекe Nginx ↔ PHP-FPM по TCP:9000.

2) Цель проекта
- Собрать учебный стек на Docker: Nginx (reverse proxy и статика), PHP-FPM 8.4 (FastCGI на TCP:9000), PostgreSQL, pgAdmin.
- Сохранить удобство dev-опыта: Makefile, Xdebug-оверлей, healthchecks, «говорящие» конфиги и простая документация.

3) Итоговая архитектура (целевое состояние)
- PHP-FPM:
    - Выполняет PHP.
    - Слушает TCP:9000 (FastCGI).
- Nginx:
    - Отдаёт статику из DocumentRoot.
    - Проксирует .php в PHP-FPM по сети (fastcgi_pass php-fpm:9000).
- PostgreSQL и pgAdmin:
    - PostgreSQL доступен на 5432, данные в именованном томе.
    - pgAdmin — веб-интерфейс для управления PostgreSQL.
- Внешние порты:
    - Nginx: 80 → localhost:80.
    - pgAdmin: 80 контейнера → localhost:8080.
    - PostgreSQL: 5432 → localhost:5432.

4) Ключевые решения и ограничения
- Соединение Nginx ↔ PHP-FPM: строго TCP по адресу контейнера PHP-FPM и порту 9000.
- Простота и обучаемость в приоритете над прод-хардненингом.
- Проект не для продакшена; безопасности достаточно на уровне локальной среды.

5) Правила для AI Assistant и Junie
- Язык общения и документации: русский.
- Бэкенд-стек: PHP 8.4 (FPM), Nginx (stable), PostgreSQL (актуальная LTS/стабильная, например 16), pgAdmin 4, Docker Compose v2+.
- Не менять структуру проекта без отдельного обсуждения.
- Предпочитать минимальные и точечные изменения с чёткими комментариями и контекстом.
- Для правок кода/конфигов использовать формат фрагментов с контекстом и пометкой «// ... existing code ...».
- Не упоминать содержимое вложений/аттачей без необходимости.
- Не превращать проект в прод: без сложных CI/CD и enterprise-практик.

6) Именование и структура репозитория
- Имя репозитория (предложение): php-nginx-tcp.
- Базовая структура:
    - config/
        - nginx/nginx.conf (проксирование в PHP-FPM по TCP:9000)
        - php/php.ini (dev-настройки, Xdebug через env)
    - docker/
        - php.Dockerfile (база для php-fpm, EXPOSE 9000 для метки; наружу не пробрасывать отдельно)
    - env/
        - .env.example (переменные PostgreSQL, pgAdmin, Xdebug и т.д.)
    - public/ (DocumentRoot)
    - docker-compose.yml (Nginx, PHP-FPM, PostgreSQL, pgAdmin)
    - docker-compose.xdebug.yml (оверлей для Xdebug)
    - Makefile (команды управления стеком)
    - README.md (инструкции по запуску и конфигурации)
    - docs/AI-CONTEXT.md (этот файл)

7) План реализации (пошагово)
- Шаг 1: Инициализировать репозиторий и базовую структуру (см. выше).
- Шаг 2: Настроить PHP-FPM:
    - listen = 9000 (tcp), pm = dynamic/static по умолчанию для dev.
    - Установить fcgi (cgi-fcgi) для healthcheck.
- Шаг 3: Настроить Nginx:
    - Статика: root /var/www/html.
    - .php: fastcgi_pass php-fpm:9000; корректные fastcgi_param SCRIPT_FILENAME и DOCUMENT_ROOT.
    - Включить index.php в index.
- Шаг 4: Настроить PostgreSQL и том для данных.
- Шаг 5: Настроить pgAdmin:
    - Публиковать на 8080.
    - ENV: PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD.
- Шаг 6: Healthchecks:
    - PHP-FPM: cgi-fcgi -bind -connect 127.0.0.1:9000.
    - Nginx: HTTP GET http://localhost/.
    - PostgreSQL: pg_isready -U $POSTGRES_USER -h 127.0.0.1.
    - pgAdmin: HTTP GET http://localhost:8080.
- Шаг 7: depends_on:
    - Nginx ждёт healthy PHP-FPM.
    - pgAdmin может стартовать независимо; подключение к БД задаётся вручную из UI.
- Шаг 8: Запуск и проверка:
    - make up → http://localhost, phpinfo, тест .php.
    - Проверить логи и здоровье контейнеров.
- Шаг 9: README.md:
    - Описать архитектуру, переменные окружения, быстрый старт.
- Шаг 10: Xdebug:
    - Проверить подключение к IDE (порт 9003).

8) Требования к конфигурации PHP-FPM (целевое)
- listen: 0.0.0.0:9000 (или просто 9000 внутри контейнера).
- user/group: www-data (или по умолчанию образа).
- Разрешить нужные расширения: pdo, pdo_pgsql, pgsql, mbstring, xml, gd, bcmath, zip и т.п.
- В образе установить fcgi (cgi-fcgi) для healthcheck.
- Не пробрасывать порт 9000 наружу напрямую; доступ к нему — только внутри сети Docker.

9) Требования к конфигурации Nginx (целевое)
- root: /var/www/html.
- index: index.php index.html.
- fastcgi_pass: php-fpm:9000 (имя сервиса PHP-FPM в docker-compose).
- Обязательные fastcgi_param:
    - SCRIPT_FILENAME $document_root$fastcgi_script_name
    - DOCUMENT_ROOT $document_root
- Включить стандартные fastcgi_params и fastcgi_buffers/timeout для dev по умолчанию.
- client_max_body_size согласовать с PHP upload_max_filesize/post_max_size.

10) PostgreSQL и pgAdmin
- PostgreSQL:
    - Порт 5432 → localhost:5432.
    - Данные: именованный том (например, postgres-data).
    - ENV: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB.
- pgAdmin:
    - Порт 80 контейнера → localhost:8080.
    - ENV: PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD.
    - Подключение к БД — через UI: host=postgres, port=5432.

11) Общие тома и монтирование
- Код (public/) монтируется в Nginx и PHP-FPM по одному и тому же пути (обычно /var/www/html) — изменения видны сразу.
- Конфиги монтируются только на чтение.
- Данные PostgreSQL — в отдельном томе (persist).

12) Healthchecks (целевое)
- PHP-FPM: cgi-fcgi -bind -connect 127.0.0.1:9000.
- Nginx: curl -fsS http://localhost/ или wget -q -O- http://localhost/.
- PostgreSQL: pg_isready -U $POSTGRES_USER -h 127.0.0.1.
- pgAdmin: HTTP-запрос к http://localhost:8080/.
- depends_on: Nginx ждёт healthy PHP-FPM.

13) Makefile и команды
- Сохранить привычные команды: setup, up, down, restart, logs, status.
- xdebug-up / xdebug-down — запуск с оверлеем Xdebug.
- clean / clean-all — удаление контейнеров/томов (с предупреждениями).

14) Xdebug (заметки)
- Версия Xdebug 3; порт клиента IDE — 9003.
- Управление через ENV (XDEBUG_MODE, XDEBUG_START, XDEBUG_CLIENT_HOST=host.docker.internal).
- Транспорт Xdebug не зависит от способа связи Nginx↔FPM (TCP:9000).

15) Тестирование и критерии приёмки
- http://localhost открывается; статика и .php отдаются.
- phpinfo() отражает fpm и xdebug (если включён).
- Логи Nginx и PHP-FPM без критических ошибок.
- Healthchecks всех сервисов OK.
- pgAdmin доступен на http://localhost:8080; можно подключиться к PostgreSQL (host=postgres).
- В public/ изменения видны сразу.

16) Не-цели
- Продакшен-хардненинг, SSL/TLS, балансировка, CDN.
- Сложная оркестрация (Swarm/Kubernetes).
- Тюнинг производительности под высокую нагрузку.

17) Безопасность (в рамках учебного проекта)
- Не хранить реальные секреты в репозитории.
- .env — локально; .env.example — шаблон без секретов.
- Порты открывать только локально; не публиковать наружу без необходимости.
- Пароли в примерах — слабые и для dev, меняйте локально.

18) Версионирование и совместимость
- PHP 8.4, Nginx stable, PostgreSQL 16 (или актуальная стабильная), pgAdmin 4.
- Docker Compose v2+.
- По возможности использовать alpine-образы, держать образы компактными.
- Согласованность путей между Nginx и PHP-FPM (DOCUMENT_ROOT, SCRIPT_FILENAME).

19) Потенциальные риски и их обработка
- Неправильный SCRIPT_FILENAME/DOCUMENT_ROOT → 404/502 на .php.
    - Сверить fastcgi_param и пути в обоих контейнерах.
- Несоответствие client_max_body_size и upload_max_filesize/post_max_size → ошибки загрузки.
    - Согласовать значения в Nginx и PHP.
- Порт 9000 перекрыт или опубликован наружу → утечка поведенческой поверхности.
    - Не публиковать 9000, использовать только внутреннюю сеть Docker.
- pgAdmin не подключается к БД:
    - Проверить сеть docker, host=postgres, port=5432, креды env.
- Xdebug не подключается:
    - Проверить порт 9003, XDEBUG_MODE/START, client_host=host.docker.internal.

20) Что подготовить перед реализацией
- Утвердить пути: DocumentRoot=/var/www/html одинаково в Nginx и PHP-FPM.
- Убедиться, что fastcgi_pass будет указывать на сервис PHP-FPM: php-fpm:9000.
- Определить минимальный набор PHP-расширений (включая pdo_pgsql/pgsql).
- Определить переменные окружения в .env.example:
    - POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
    - PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD
    - XDEBUG_MODE, XDEBUG_START, XDEBUG_CLIENT_HOST
- Спланировать healthchecks и depends_on.

Конец документа.