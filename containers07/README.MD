# Лабораторная работа №6: Создание многоконтейнерного приложения

## Цель работы
Ознакомиться с работой многоконтейнерного приложения на базе docker-compose.

## Задание
Создать PHP приложение на базе трех контейнеров: nginx, php-fpm, mariadb, используя docker-compose.

## Выполнение работы

### 1. Настройка конфигурационных файлов

В .gitignore добавил исключение для содержимого папки mounts/site:

    # Ignore files and directories
    mounts/site/*

Создал конфигурацию Nginx в nginx/default.conf:

    server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.php;
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    location ~ \.php$ {
        fastcgi_pass backend:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}

Создал docker-compose.yml с тремя сервисами:

    version: '3.9'

services:
  frontend:
    image: nginx:1.19
    volumes:
      - ./mounts/site:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
    networks:
      - internal
  backend:
    image: php:7.4-fpm
    volumes:
      - ./mounts/site:/var/www/html
    networks:
      - internal
    env_file:
      - mysql.env
  database:
    image: mysql:8.0
    env_file:
      - mysql.env
    networks:
      - internal
    volumes:
      - db_data:/var/lib/mysql

networks:
  internal: {}

volumes:
  db_data: {}

Создал файл mysql.env с переменными окружения для MySQL:

MYSQL_ROOT_PASSWORD=secret
MYSQL_DATABASE=app
MYSQL_USER=user
MYSQL_PASSWORD=secret

### 2. Запуск и тестирование

Запустил контейнеры командой:

    docker-compose up -d

Проверил работу сайта по адресу http://localhost. После перезагрузки страницы сайт отобразился корректно.

### 2. Ответы на вопросы

1. В каком порядке запускаются контейнеры?

    Контейнеры запускаются в порядке, определенном их зависимостями. В нашем случае сначала запускается database, затем backend, и только потом frontend, так как backend зависит от database, а frontend зависит от backend.

2. Где хранятся данные базы данных?

    Данные базы данных хранятся в томе db_data, который монтируется в /var/lib/mysql контейнера database. Этот том сохраняет данные даже после остановки контейнеров.

3. Как называются контейнеры проекта?

    По умолчанию контейнеры получают имена в формате <имя_папки_проекта>_<имя_сервиса>_<номер>. В нашем случае:

    containers06_frontend_1
    containers06_backend_1
    containers06_database_1

4. Как добавить файл app.env с переменной окружения APP_VERSION?

    Необходимо:
    Создать файл app.env с содержимым:
    APP_VERSION=1.0
    Изменить docker-compose.yml, добавив app.env в секции env_file для нужных сервисов:
        frontend:
    env_file:
        - app.env
    backend:
    env_file:
        - mysql.env
        - app.env

### Выводы

В ходе лабораторной работы я научился:

    1. Создавать многоконтейнерные приложения с помощью docker-compose
    2. Настраивать взаимодействие между контейнерами (nginx + php-fpm + mysql)
    3. Управлять переменными окружения в docker-compose
    4. Работать с томами для сохранения данных БД
    5. Настраивать сетевые взаимодействия между контейнерами



