<!-- ---
title: 'Development Environment'
description: 'Set up a stable environment for package development. Starting with installing Composer, configuring package details and PSR autoloading in composer.json to pulling in the package locally and testing with Orchestra Testbench.'
tags: ['development setup', 'composer', 'package skeleton', 'PSR', 'namespacing', 'testing', 'testbench']
image: 'https://www.laravelpackage.com/assets/pages/laravelpackage.jpeg'
date: 2019-09-17
--- -->

# Разработка пакетов для Laravel · Среда разработки

## Установка Composer

Есть большая вероятность, что у вас уже установлен Composer. Однако, если вы еще не установили Composer, то самый быстрый способ начать работу – скопировать сценарий, представленный на странице загрузки [Composer](https://getcomposer.org/download/). При копировании и вставке предоставленного сценария в вашу командную строку установщик `composer.phar` будет загружен, запущен и снова удален. Вы можете проверить успешность установки, запустив `composer --version`. Чтобы обновить Composer до последней версии, запустите `composer self-update`.

## Каркас пакета

Чтобы начать разработку пакета, сначала создайте пустой каталог. Нет необходимости вкладывать пакеты в существующий проект Laravel. Я настоятельно рекомендую организовывать ваши пакеты отдельно от ваших проектов Laravel.

Например, я храню все пакеты в `~/packages/`, а мои приложения Laravel «живут» в `~/websites/`.

## Composer.json

Начнем с создания файла `composer.json` в корне каталога вашего пакета, имеющего минимальную конфигурацию (как показано ниже). Замените все детали из примера своими.

Лучше всего следовать соглашению при именовании ваших пакетов. Стандартное соглашение – использовать ваше имя пользователя GitHub / Gitlab / Bitbucket / и т. д., за которым следует косая черта «/», а затем имя вашего пакета в регистре `kebab-case`.

Пример `composer.json` указан ниже:

```json
{
  "name": "johndoe/blogpackage",
  "description": "A demo package",
  "type": "library",
  "license": "MIT",
  "authors": [
    {
      "name": "John Doe",
      "email": "john@doe.com"
    }
  ],
  "require": {}
}
```

В качестве альтернативы вы можете создать свой файл `composer.json`, используя команду `composer init` в пустом каталоге вашего пакета.

Если вы планируете **опубликовать** пакет, то важно выбрать подходящий тип пакета (в нашем случае `library`) и лицензию, например, «MIT». Узнайте больше о лицензиях с открытым исходным кодом на сайте [ChooseALicense.com](https://choosealicense.com/).

## Пространство имен

Поскольку мы хотим использовать каталог `src/` для хранения нашего кода, нам нужно указать Composer сопоставить пространство имен пакета с этим конкретным каталогом при создании автозагрузчика (`vendor/autoload.php`).

Мы можем зарегистрировать наше пространство имен под ключом `psr-4` ключа `autoload` файла `composer.json` следующим образом (замените пространство имен своим собственным):

```json
{
  ...,

  "require": {},

  "autoload": {
    "psr-4": {
      "JohnDoe\\BlogPackage\\": "src"
    }
  }
}
```

## Автозагрузка PSR-4

Теперь вы можете задаться вопросом, зачем нам нужен ключ `psr-4`. PSR – рекомендации по стандартам PHP, разработанные [группой совместимости фреймворков PHP](https://www.php-fig.org/) (PHP-FIG). Эта группа из 20 участников, представляющих разные слои сообщества PHP, предложила [серию PSR](https://www.php-fig.org/psr/).

PSR-4 представляет собой рекомендацию относительно автозагрузки классов с учетом путей к файлам, заменив существовавший до того времени [стандарт автозагрузки PSR-0](https://www.php-fig.org/psr/psr-0/).

Существенная разница между PSR-0 и PSR-4 заключается в том, что PSR-4 позволяет сопоставить базовый каталог с частью пространства имени и, следовательно, допускает более короткие пространства имен. Я думаю, что этот [комментарий](https://stackoverflow.com/questions/24868586/what-are-the-differences-between-psr-0-and-psr-4/50226226#50226226) на StackOverflow содержит четкое описание того, как PSR-0 и PSR-4 работают.

PSR-0

```json
"autoload": {
    "psr-0": {
        "Book\\": "src/",
        "Vehicle\\": "src/"
    }
}
```

- Искать `Book\History\UnitedStates` в `src/Book/History/UnitedStates.php`

- Искать `Vehicle\Air\Wings\Airplane` в `src/Vehicle/Air/Wings/Airplane.php`

PSR-4

```json
"autoload": {
    "psr-4": {
        "Book\\": "src/",
        "Vehicle\\": "src/"
    }
}
```

- Искать `Book\History\UnitedStates` в `src/History/UnitedStates.php`

- Искать `Vehicle\Air\Wings\Airplane` в `src/Air/Wings/Airplane.php`

## Локальный импорт пакета

В помощь с разработкой, вам может потребоваться локальный пакет в локальном проекте Laravel.

Если у вас есть локальный проект Laravel, то вы можете добавить в зависимость свой пакет локально, определив собственный так называемый «репозиторий» в файле `composer.json` **вашего приложения Laravel**.

Добавьте следующий ключ `repositories` под ключом `scripts` файла `composer.json` вашего приложения Laravel, заменив значение `url` на каталог, в котором находится ваш пакет:

```json
{
  "scripts": { ... },

  "repositories": [
    {
      "type": "path",
      "url": "../../packages/blogpackage"
    }
  ]
}
```

Теперь вы можете включить свой локальный пакет в приложении Laravel, используя выбранное пространство имен пакета. Следуя нашему примеру, это будет:

```bash
composer require johndoe/blogpackage
```

Если у вас есть несколько пакетов в одном каталоге и вы хотите, чтобы Composer выполнял поиск всех, то вы можете указать расположение пакета, используя метасимвол подстановки `*` следующим образом:

```json
{
  "scripts": { ... },

  "repositories": [
    {
      "type": "path",
      "url": "../../packages/*"
    }
  ]
}
```


**Важно:** вам нужно будет выполнять команду `composer update` в вашем приложении Laravel всякий раз, когда вы вносите изменения в файл `composer.json` вашего пакета или любых поставщиков, которые он регистрирует.

## Пакет Orchestra Testbench

Теперь у нас есть файл `composer.json` и пустой каталог `src/`. Однако у нас нет доступа к функционалу Laravel компонентов `Illuminate`.

Чтобы использовать эти компоненты в нашем пакете, нам потребуется [Orchestra Testbench](https://github.com/orchestral/testbench). Обратите внимание, что каждая версия фреймворка Laravel имеет соответствующую версию Orchestra Testbench. В этом разделе я предполагаю, что мы разрабатываем пакет для **Laravel 8.x**, который является последней версией на момент написания этого раздела.

```bash
composer require --dev "orchestra/testbench=^6.0"
```

Полная таблица совместимости Orchestra Testbench, взятая из [исходной документации](https://github.com/orchestral/testbench), приведена ниже:


Фреймворк Laravel  | Пакет Testbench
:---------|:----------
8.x      | 6.x
7.x      | 5.x
6.x      | 4.x
5.8.x    | 3.8.x
5.7.x    | 3.7.x
5.6.x    | 3.6.x
5.5.x    | 3.5.x
5.4.x    | 3.4.x
5.3.x    | 3.3.x
5.2.x    | 3.2.x
5.1.x    | 3.1.x
5.0.x    | 3.0.x

После установки Orchestra Testbench вы найдете каталог `vendor/orchestra/testbench-core`, содержащий каталоги `laravel` и `src`. Каталог `laravel` напоминает структуру реального приложения Laravel, а каталог `src` содержит помощников Laravel, обеспечивающих взаимодействие со структурой каталогов проекта, например, связанные с манипуляциями с файлами.

Перед каждым тестом TestBench создает тестовое окружение полностью загруженного (тестового) приложения. Если мы используем базовый класс `TestCase` Orchestra TestBench для наших тестов, то за создание этого тестового приложения будут отвечать методы, предоставляемые трейтом `CreatesApplication` в пространстве имен `Orchestra\Testbench\Concerns`. Если мы посмотрим на один из этих методов, `getBasePath()`, то мы увидим, что он напрямую указывает на папку `laravel` пакета Orchestra Testbench.

```php
// 'vendor/orchestra/testbench-core/src/Concerns/CreatesApplication.php'
/**
 * Get base path.
 *
 * @return string
 */
protected function getBasePath()
{
    return \realpath(__DIR__.'/../../laravel');
}
```
