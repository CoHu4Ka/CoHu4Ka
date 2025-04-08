# Лабораторная работа: Управление взаимодействием нескольких контейнеров

## Цель работы
Научиться управлять взаимодействием нескольких Docker-контейнеров в рамках одного приложения.

## Задание
Создать PHP-приложение на базе двух контейнеров:
- Nginx (frontend)
- PHP-FPM (backend)

## Выполнение работы

### 1. Конфигурационные файлы

#### nginx/default.conf
``nginx
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

### 2. Запуск контейнеров
Создаем Docker-сеть:
docker network create internal

Запускаем контейнер PHP-FPM (backend):
docker run -d --name backend \
  --network internal \
  -v $(pwd)/mounts/site:/var/www/html \
  php:7.4-fpm

Запускаем контейнер Nginx (frontend):
docker run -d --name frontend \
  --network internal \
  -v $(pwd)/mounts/site:/var/www/html \
  -v $(pwd)/nginx/default.conf:/etc/nginx/conf.d/default.conf \
  -p 80:80 \
  nginx:1.23-alpine

### 3. Проверка работы
Приложение доступно по адресу: http://localhost


### Ответы на контрольные вопросы

#### Взаимодействие контейнеров
Контейнеры взаимодействуют через Docker-сеть internal. Nginx передает PHP-запросы в контейнер php-fpm по имени backend на порт 9000.

#### Разрешение имен в сети internal
Docker предоставляет встроенный DNS-сервер, который разрешает имена контейнеров в их IP-адреса внутри сети. Контейнеры могут обращаться друг к другу по именам (frontend и backend).

#### Необходимость переопределения конфигурации Nginx
Стандартная конфигурация Nginx не содержит:

1. Настроек для обработки PHP-файлов
2. Указания на PHP-FPM сервер
3. Правильных параметров fastcgi
4. Корректных путей к файлам приложения

### Выводы
В результате работы:

1. Создана сетевая инфраструктура для многоконтейнерного приложения
2. Настроено взаимодействие между веб-сервером и обработчиком PHP
3. Отработаны навыки работы с Docker-сетями и пробросом конфигураций
4. Получен практический опыт развертывания PHP-приложений в контейнерах
5. Проект демонстрирует основные принципы организации микросервисной архитектуры с использованием Docker.