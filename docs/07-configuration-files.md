<!-- ---
title: "Package Configuration"
description: "Nearly all packages include a certain configuration to allow easy modification by the end-user. This section explains how to create a config file and publish this configuration within a Laravel project."
tags: ["Configuration", "Publishing Configuration"]
image: "https://www.laravelpackage.com/assets/pages/laravelpackage.jpeg"
date: 2019-09-17
--- -->

# Разработка пакетов для Laravel · Конфигурационные файлы

Вполне вероятно, что ваш пакет допускает конфигурирование конечным пользователем.

Если вы хотите предложить собственные параметры конфигурации, создайте новый каталог `config` в корне пакета и добавьте файл с именем `config.php`, который возвращает массив параметров:

```php
// 'config/config.php'
<?php

return [
  'posts_table' => 'posts',
  // Другие параметры ...
];
```

## Слияние с существующей конфигурацией

После регистрации конфигурационного файла в методе `register()` нашего поставщика служб под определенным «ключом» (`blogpackage` в нашем случае), мы можем получить доступ к значениям конфигурации из глобального помощника `config`, добавив префикс нашего «ключа» следующим образом: `config('blogpackage.posts_table')`.

```php
// 'BlogPackageServiceProvider.php'
public function register()
{
  $this->mergeConfigFrom(__DIR__.'/../config/config.php', 'blogpackage');
}
```

## Публикация конфигурационного файла

Чтобы пользователи могли изменять значения конфигурации по умолчанию, нам необходимо предоставить им возможность экспортировать конфигурационный файл. Мы можем зарегистрировать все «публикуемые» ресурсы в методе `boot()` поставщика служб пакета. Поскольку мы хотим использовать функционал публикации только при загрузке из консоли, то сначала мы проверим, что текущее приложение запущено из консоли. Мы зарегистрируем публикуемый файл конфигурации под тегом `config`, передав второй параметр методу `publishes()`.

```php
// 'BlogPackageServiceProvider.php'
public function boot()
{
  if ($this->app->runningInConsole()) {

    $this->publishes([
      __DIR__.'/../config/config.php' => config_path('blogpackage.php'),
    ], 'config');

  }
}
```

Конфигурационный файл `blogpackage.php` пакета можно экспортировать в каталог `/config` проекта Laravel, используя команду, указанную ниже:

```bash
php artisan vendor:publish --provider="JohnDoe\BlogPackage\BlogPackageServiceProvider" --tag="config"
```
