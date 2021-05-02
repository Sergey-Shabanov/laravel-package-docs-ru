<!-- ---
title: "Testing"
description: "Testing is an essential part of every package to ensure proper behavior and allow refactoring with confidence. This section explains how to set up a testing environment using PHPUnit to create robust packages."
tags: ["Testing", "PHPUnit", "Directory Structure"]
image: "https://www.laravelpackage.com/assets/pages/laravelpackage.jpeg"
date: 2019-09-17
--- -->

# Разработка пакетов для Laravel · Тестирование

Очень важно иметь надлежащее тестовое покрытие для кода, предоставляемого пакетом. Добавление тестов в наш пакет может подтвердить поведение существующего кода, убедиться, что все по-прежнему работает при добавлении новых функций, и гарантировать, что мы сможем безопасно реорганизовать наш пакет на более позднем этапе.

Кроме того, хорошее покрытие кода может мотивировать потенциальных участников, предоставляя им больше уверенности в том, что их вклад не нарушит что-то еще в пакете. Тесты также позволяют другим разработчикам понять, как следует использовать определенные функции вашего пакета, и придадут им уверенности в надежности вашего пакета.

## Установка PHPUnit

Есть много вариантов для тестирования поведения в PHP. Однако мы будем придерживаться стандарта Laravel, который использует отличный инструмент PHPUnit.

Установим PHPUnit как зависимость `dev` в нашем пакете:

```bash
composer require --dev phpunit/phpunit
```

> **Примечание:** вам может потребоваться установить определенную версию, если вы разрабатываете пакет для более старой версии Laravel.

Чтобы сконфигурировать PHPUnit, создайте файл `phpunit.xml` в корневом каталоге пакета.
Затем скопируйте следующий шаблон, чтобы использовать базу данных sqlite, хранимую в памяти, и включить красочные отчеты.

`phpunit.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  bootstrap="vendor/autoload.php"
  backupGlobals="false"
  backupStaticAttributes="false"
  colors="true"
  verbose="true"
  convertErrorsToExceptions="true"
  convertNoticesToExceptions="true"
  convertWarningsToExceptions="true"
  processIsolation="false"
  stopOnFailure="false"
  xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.3/phpunit.xsd"
>
  <coverage>
    <include>
      <directory suffix=".php">src/</directory>
    </include>
  </coverage>
  <testsuites>
    <testsuite name="Unit">
      <directory suffix="Test.php">./tests/Unit</directory>
    </testsuite>
    <testsuite name="Feature">
      <directory suffix="Test.php">./tests/Feature</directory>
    </testsuite>
  </testsuites>
  <php>
    <env name="DB_CONNECTION" value="testing"/>
    <env name="APP_KEY" value="base64:2fl+Ktvkfl+Fuz4Qp/A75G2RTiWVA/ZoKZvp6fiiM10="/>
  </php>
</phpunit>
```

Обратите внимание на фиктивный `APP_KEY` в приведенном выше примере. Эта переменная окружения используется [шифровальщиком Laravel](https://github.com/russsiq/laravel-docs-8.x-ru/blob/main/docs/encryption.md#using-the-encrypter), который может использоваться в наших тестах. В большинстве случаев фиктивного значения будет достаточно. Однако вы можете либо изменить это значение, чтобы отразить фактический ключ приложения (вашего приложения Laravel), либо полностью оставить его, если ваш набор тестов не взаимодействует с шифровальщиком.

## Структура каталогов

Для размещения функциональных и модульных тестов создайте каталог `tests/` с подкаталогами `Unit` и `Feature` и базовым файлом `TestCase.php`. Структура выглядит следующим образом:

```json
- tests
    - Feature
    - Unit
      TestCase.php
```

`TestCase.php` расширяет `\Orchestra\Testbench\TestCase` (см. пример ниже) и содержит задачи, связанные с настройкой нашего «мира» перед выполнением каждого теста. В классе `TestCase` мы реализуем три важных конфигурационных метода:

- `getPackageProviders()`
- `getEnvironmentSetUp()`
- `setUp()`

Давайте рассмотрим эти методы один за другим.

### Метод `setUp()`

Возможно, вы уже использовали этот метод в своих тестах. Часто его используют, когда нужна определенная модель во всех следующих тестах. Таким образом, экземпляр этой модели может быть извлечен в методе `setUp()`, который вызывается перед каждым тестом. В рамках тестов желаемую модель можно получить из переменной экземпляра класса `Test`. При использовании этого метода не забудьте вызвать родительский метод `setUp()`, и не забудьте вернуть `void`.

---

### Метод `getEnvironmentSetUp()`

Согласно документации Orchestra Testbench: «Если вам нужно добавить что-то на раннем этапе процесса начальной загрузки приложения, то вы можете использовать метод `getEnvironmentSetUp()`». Поэтому я предлагаю вызывать его перед методом `setUp()`.

---

### Метод `getPackageProviders()`

Как следует из названия, мы можем загрузить наших поставщиков служб в методе `getPackageProviders()`. Мы сделаем это, вернув массив, содержащий всех провайдеров. На данный момент мы просто включим поставщика для конкретного пакета, но представьте, что если пакет использует `EventServiceProvider`, мы также должны зарегистрировать его здесь.

---

В пакете `TestCase` наследует от `Orchestra\Testbench\TestCase`:

```php
// 'tests/TestCase.php'
<?php

namespace JohnDoe\BlogPackage\Tests;

use JohnDoe\BlogPackage\BlogPackageServiceProvider;

class TestCase extends \Orchestra\Testbench\TestCase
{
  public function setUp(): void
  {
    parent::setUp();
    // Дополнительная установка ...
  }

  protected function getPackageProviders($app)
  {
    return [
      BlogPackageServiceProvider::class,
    ];
  }

  protected function getEnvironmentSetUp($app)
  {
    // Выполнить установку окружения ...
  }
}
```

Прежде чем мы сможем запустить набор тестов PHPUnit, нам сначала нужно сопоставить пространство имен тестирования с соответствующим каталогом в ​​файле `composer.json` под ключом `autoload-dev` -> `psr-4`:

```json
{
  ...,

  "autoload": {},

  "autoload-dev": {
    "psr-4": {
      "JohnDoe\\BlogPackage\\Tests\\": "tests"
    }
  }
}
```

Наконец, выполните `composer dump-autoload` для перестроения файла автозагрузки.
