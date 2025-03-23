# Лабораторная работа №5: Запуск сайта в контейнере

## Цель работы
Целью данной работы является подготовка образа контейнера для запуска веб-сайта на базе Apache HTTP Server, PHP (mod_php) и MariaDB.

## Задание
1. Создать Dockerfile для сборки образа контейнера.
2. Установить WordPress и проверить его работоспособность.
3. Настроить монтирование тома для базы данных MariaDB.
4. Обеспечить доступ к серверу по порту 8000.

## Выполнение работы
# 1. Подготовка

    Я убедился, что на моем компьютере установлен Docker.

    Создал репозиторий containers05 и склонировал его на локальный компьютер.

# 2. Создание структуры папок и файлов

    В папке containers05 я создал папку files, а внутри неё подпапки:

    files/apache2 — для файлов конфигурации Apache.

    files/php — для файлов конфигурации PHP.

    files/mariadb — для файлов конфигурации MariaDB.

    files/supervisor — для файла конфигурации Supervisor.

# 3. Создание Dockerfile

    Я создал файл Dockerfile в папке containers05 со следующим содержимым:

    Используем образ Debian
    FROM debian:latest

    Устанавливаем необходимые пакеты
    RUN apt-get update && \
        apt-get install -y apache2 php libapache2-mod-php php-mysql mariadb-server supervisor && \
        apt-get clean

    Монтируем тома для данных MySQL и логов
    VOLUME /var/lib/mysql
    VOLUME /var/log

    Копируем файлы WordPress
    ADD https://wordpress.org/latest.tar.gz /var/www/html/

    Копируем конфигурационные файлы
    COPY files/apache2/000-default.conf /etc/apache2/sites-available/000-default.conf
    COPY files/apache2/apache2.conf /etc/apache2/apache2.conf
    COPY files/php/php.ini /etc/php/8.2/apache2/php.ini
    COPY files/mariadb/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf
    COPY files/supervisor/supervisord.conf /etc/supervisor/supervisord.conf

    Создаем папку для сокета MySQL и устанавливаем права
    RUN mkdir /var/run/mysqld && chown mysql:mysql /var/run/mysqld

    Открываем порт 80
    EXPOSE 80

    Запускаем Supervisor
    CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/supervisord.conf"]

# 4. Извлечение конфигурационных файлов

    Я собрал образ контейнера:
    docker build -t apache2-php-mariadb .

    Запустил контейнер:
    docker run -d --name apache2-php-mariadb apache2-php-mariadb

    Скопировал конфигурационные файлы из контейнера:
    docker cp apache2-php-mariadb:/etc/apache2/sites-available/000-default.conf files/apache2/
    docker cp apache2-php-mariadb:/etc/apache2/apache2.conf files/apache2/
    docker cp apache2-php-mariadb:/etc/php/8.2/apache2/php.ini files/php/
    docker cp apache2-php-mariadb:/etc/mysql/mariadb.conf.d/50-server.cnf files/mariadb/

    Остановил и удалил контейнер:
    docker stop apache2-php-mariadb
    docker rm apache2-php-mariadb

# 5. Настройка конфигурационных файлов
    Apache2:

    В файле files/apache2/000-default.conf я заменил строку #ServerName www.example.com на ServerName localhost.

    Заменил ServerAdmin webmaster@localhost на свой email.

    Добавил строку DirectoryIndex index.php index.html после DocumentRoot /var/www/html.

    В файле files/apache2/apache2.conf добавил строку ServerName localhost в конце файла.

    PHP:

    В файле files/php/php.ini я заменил строку ;error_log = php_errors.log на error_log = /var/log/php_errors.log.

    Настроил параметры:
    memory_limit = 128M
    upload_max_filesize = 128M
    post_max_size = 128M
    max_execution_time = 120
    MariaDB:

    В файле files/mariadb/50-server.cnf я раскомментировал строку #log_error = /var/log/mysql/error.log.

# 6. Создание скрипта запуска Supervisor

    Я создал файл files/supervisor/supervisord.conf со следующим содержимым:
    [supervisord]
    nodaemon=true
    logfile=/dev/null
    user=root

    [program:apache2]
    command=/usr/sbin/apache2ctl -D FOREGROUND
    autostart=true
    autorestart=true
    startretries=3
    stderr_logfile=/proc/self/fd/2
    user=root

    [program:mariadb]
    command=/usr/sbin/mariadbd --user=mysql
    autostart=true
    autorestart=true
    startretries=3
    stderr_logfile=/proc/self/fd/2
    user=mysql

# 7. Сборка и запуск контейнера

    Я пересобрал образ:
    docker build -t apache2-php-mariadb .

    Запустил контейнер:
    docker run -d -p 8000:80 --name apache2-php-mariadb apache2-php-mariadb

# 8. Создание базы данных и пользователя

    Я подключился к контейнеру:
    docker exec -it apache2-php-mariadb bash

    Создал базу данных и пользователя:
    mysql
    CREATE DATABASE wordpress;
    CREATE USER 'wordpress'@'localhost' IDENTIFIED BY 'wordpress';
    GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'localhost';
    FLUSH PRIVILEGES;
    EXIT;

# 9. Настройка WordPress

    Я открыл браузер и перешел по адресу http://localhost:8000/.

    Указал параметры подключения к базе данных:

    Имя базы данных: wordpress

    Имя пользователя: wordpress

    Пароль: wordpress

    Адрес сервера базы данных: localhost

    Префикс таблиц: wp_

    Скопировал содержимое файла конфигурации в файл files/wp-config.php.

# 10. Добавление файла конфигурации WordPress в Dockerfile

    Я добавил в Dockerfile строку:
    COPY files/wp-config.php /var/www/html/wordpress/wp-config.php

# 11. Пересборка и тестирование

    Я пересобрал образ и запустил контейнер:
    docker build -t apache2-php-mariadb .
    docker run -d -p 8000:80 --name apache2-php-mariadb apache2-php-mariadb
    Проверил работоспособность сайта WordPress.

## Ответы на вопросы
1. Какие файлы конфигурации были изменены?
    000-default.conf, apache2.conf, php.ini, 50-server.cnf.

2. За что отвечает инструкция DirectoryIndex в файле конфигурации apache2?
    Указывает, какие файлы должны быть использованы в качестве индексных при запросе к директории.

3. Зачем нужен файл wp-config.php?
    Он содержит настройки подключения к базе данных и другие важные параметры для работы WordPress.

4. За что отвечает параметр post_max_size в файле конфигурации php?
    Определяет максимальный размер данных, которые могут быть отправлены через POST-запрос.

5. Укажите, на ваш взгляд, какие недостатки есть в созданном образе контейнера?
    Отсутствие разделения контейнеров для Apache, PHP и MariaDB, что может усложнить масштабирование и управление.

## Выводы
В ходе выполнения лабораторной работы я создал Docker-образ для запуска веб-сайта на базе Apache, PHP и MariaDB. Установил и настроил WordPress, проверил его работоспособность. Я изучил основы работы с Docker, настройку конфигурационных файлов и управление контейнерами.



