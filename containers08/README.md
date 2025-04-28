# Лабораторная работа: Настройка непрерывной интеграции с Github Actions

## Цель
Настройка CI/CD для веб-приложения на PHP с использованием:
- Github Actions
- Docker-контейнеров
- Автоматизированного тестирования

## Задание

    Создать веб-приложение на PHP с использованием SQLite
    Написать тесты для проверки функциональности приложения
    Настроить непрерывную интеграцию с помощью Github Actions
    Создать Docker-образ для развертывания приложения
    Настроить автоматическое тестирование при каждом push в репозиторий

## Описание выполнения работы

### Структура проекта
    containers08/
    ├── .github/
    │   └── workflows/
    │       └── main.yml
    ├── site/
    │   ├── modules/
    │   │   ├── database.php
    │   │   └── page.php
    │   ├── templates/
    │   │   └── index.tpl
    │   ├── styles/
    │   │   └── style.css
    │   ├── config.php
    │   └── index.php
    ├── sql/
    │   └── schema.sql
    ├── tests/
    │   ├── testframework.php
    │   └── tests.php
    ├── Dockerfile
    └── README.md

### Реализация основных компонентов

1. Класс Database (modules/database.php):

```php
<?php

class Database {
    private $db;

    public function __construct($path) {
        $this->db = new SQLite3($path);
    }

    public function Execute($sql) {
        return $this->db->exec($sql);
    }

    public function Fetch($sql) {
        $result = $this->db->query($sql);
        $data = [];
        while ($row = $result->fetchArray(SQLITE3_ASSOC)) {
            $data[] = $row;
        }
        return $data;
    }

    public function Create($table, $data) {
        $columns = implode(', ', array_keys($data));
        $values = "'" . implode("', '", array_values($data)) . "'";
        $sql = "INSERT INTO {$table} ({$columns}) VALUES ({$values})";
        $this->Execute($sql);
        return $this->db->lastInsertRowID();
    }

    public function Read($table, $id) {
        $sql = "SELECT * FROM {$table} WHERE id = {$id}";
        $result = $this->Fetch($sql);
        return $result[0] ?? null;
    }

    public function Update($table, $id, $data) {
        $updates = [];
        foreach ($data as $key => $value) {
            $updates[] = "{$key} = '{$value}'";
        }
        $updates = implode(', ', $updates);
        $sql = "UPDATE {$table} SET {$updates} WHERE id = {$id}";
        return $this->Execute($sql);
    }

    public function Delete($table, $id) {
        $sql = "DELETE FROM {$table} WHERE id = {$id}";
        return $this->Execute($sql);
    }

    public function Count($table) {
        $sql = "SELECT COUNT(*) as count FROM {$table}";
        $result = $this->Fetch($sql);
        return $result[0]['count'] ?? 0;
    }
} 
```

2. Тесты (tests/tests.php):

```php
<?php

require_once __DIR__ . '/testframework.php';

require_once __DIR__ . '/../config.php';
require_once __DIR__ . '/../modules/database.php';
require_once __DIR__ . '/../modules/page.php';

$testFramework = new TestFramework();

// Test 1: Database connection
function testDbConnection() {
    global $config;
    try {
        $db = new Database($config["db"]["path"]);
        return assertExpression($db instanceof Database, "Database connected", "Database connection failed");
    } catch (Exception $e) {
        return assertExpression(false, "", "Database connection failed: " . $e->getMessage());
    }
}

// Test 2: Test count method
function testDbCount() {
    global $config;
    $db = new Database($config["db"]["path"]);
    $count = $db->Count("page");
    return assertExpression($count == 3, "Count correct", "Count incorrect");
}

// Test 3: Test create method
function testDbCreate() {
    global $config;
    $db = new Database($config["db"]["path"]);
    $id = $db->Create("page", ["title" => "Test Page", "content" => "Test Content"]);
    return assertExpression($id > 0, "Create successful", "Create failed");
}

// Test 4: Test read method
function testDbRead() {
    global $config;
    $db = new Database($config["db"]["path"]);
    $page = $db->Read("page", 1);
    return assertExpression($page && isset($page['title']), "Read successful", "Read failed");
}

// Test 5: Test update method
function testDbUpdate() {
    global $config;
    $db = new Database($config["db"]["path"]);
    $result = $db->Update("page", 1, ["title" => "Updated Title"]);
    $page = $db->Read("page", 1);
    return assertExpression($result && $page['title'] == "Updated Title", "Update successful", "Update failed");
}

// Test 6: Test delete method
function testDbDelete() {
    global $config;
    $db = new Database($config["db"]["path"]);
    $initialCount = $db->Count("page");
    $db->Create("page", ["title" => "To Delete", "content" => "Delete me"]);
    $newCount = $db->Count("page");
    $db->Delete("page", $newCount);
    $finalCount = $db->Count("page");
    return assertExpression($initialCount == $finalCount, "Delete successful", "Delete failed");
}

// Test 7: Test Page rendering
function testPageRender() {
    global $config;
    $page = new Page(__DIR__ . '/../templates/index.tpl');
    $data = ["title" => "Test", "content" => "Test Content"];
    $rendered = $page->Render($data);
    return assertExpression(strpos($rendered, "Test Content") !== false, "Render successful", "Render failed");
}

// Add tests
$testFramework->add('Database connection', 'testDbConnection');
$testFramework->add('table count', 'testDbCount');
$testFramework->add('data create', 'testDbCreate');
$testFramework->add('data read', 'testDbRead');
$testFramework->add('data update', 'testDbUpdate');
$testFramework->add('data delete', 'testDbDelete');
$testFramework->add('page render', 'testPageRender');

// Run tests
$testFramework->run();

echo "Test results: " . $testFramework->getResult() . PHP_EOL;
```

### Ответы на вопросы

1. Что такое непрерывная интеграция?

Непрерывная интеграция (CI) - это практика разработки программного обеспечения, при которой изменения кода часто интегрируются в общую кодовую базу. Каждое изменение автоматически проверяется с помощью сборки и тестов, что позволяет быстро выявлять и исправлять проблемы.

2. Для чего нужны юнит-тесты? Как часто их нужно запускать?

Юнит-тесты нужны для:
- Проверки корректности работы отдельных модулей кода
- Быстрого выявления регрессий при изменениях
- Документирования ожидаемого поведения кода
- Упрощения рефакторинга

Юнит-тесты следует запускать как можно чаще, в идеале - при каждом изменении кода. В рамках CI они обычно запускаются при каждом push в репозиторий или создании pull request.

3. Что нужно изменить в файле .github/workflows/main.yml для того, чтобы тесты запускались при каждом создании запроса на слияние (Pull Request)?
Нужно добавить триггер pull_request:

```yaml
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
```
4. Что нужно добавить в файл .github/workflows/main.yml для того, чтобы удалять созданные образы после выполнения тестов?
Нужно добавить шаг очистки в конце job:

```yaml
- name: Clean up
  run: |
    docker rmi containers08
    docker volume rm database
```
Выводы

В ходе выполнения лабораторной работы:
- Было создано веб-приложение на PHP с использованием SQLite
- Реализованы модули для работы с базой данных и страницами
- Написаны юнит-тесты для проверки функциональности
- Настроен Docker-образ для развертывания приложения
- Реализована непрерывная интеграция с помощью Github Actions
- Настроенный процесс CI позволяет автоматически проверять каждое изменение кода, что значительно повышает надежность разработки и сокращает время на выявление ошибок.