<!-- ---
title: 'Service Providers'
description: 'The Service Provider of a package is essential to register package-specific functionality. This section will cover the role and basics of a Service Provider and explains how to create and use a Service Provider for your package.'
tags: ['Service Provider']
image: 'https://www.laravelpackage.com/assets/pages/laravelpackage.jpeg'
date: 2019-09-17
--- -->

# Разработка пакетов для Laravel · Поставщики служб

Важной частью пакета является его **Поставщик службы**. Прежде чем создавать наши собственные, я сначала объясню, что такое поставщики служб в этом разделе. Если вы уже знакомы с поставщиками служб, то перейдите к следующему разделу.

Как вы, возможно, знаете, Laravel из коробки содержит несколько поставщиков служб, а именно `AppServiceProvider`, `AuthServiceProvider`, `BroadcastServiceProvider`, `EventServiceProvider` и `RouteServiceProvider`. Эти поставщики заботятся о «начальной загрузке» / «регистрации» через связывания в контейнере служб специфичных для приложения служб, слушателей событий, посредников и маршрутов.

Каждый поставщик службы расширяет `Illuminate\Support\ServiceProvider` и реализует методы `register()` / `boot()`.

Метод `register()` используется для связывания сущностей в контейнере служб. После того, как все другие поставщики служб были зарегистрированы, то есть были вызваны все методы `register()` всех поставщиков служб, включая сторонние пакеты, тогда Laravel вызовет метод `boot()` для всех поставщиков служб.

В методе `register()` вы можете зарегистрировать связывание класса в контейнере служб, позволяя классу быть извлеченным из контейнера. Однако иногда вам нужно сослаться на другой класс, и в этом случае можно использовать `boot()`.

Вот пример того, как может выглядеть поставщик служб и какие вещи вы можете реализовать в методах `register()` и `boot()`:

```php
use App\Calculator;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
  public function register()
  {
    // Регистрация класса в контейнере служб
    $this->app->bind('calculator', function ($app) {
      return new Calculator();
    });
  }

  public function boot()
  {
    // Регистрация макрокоманды, расширяющей класс `Illuminate\Collection`
    Collection::macro('rejectEmptyFields', function () {
      return $this->reject(function ($entry) {
        return $entry === null;
       });
    });

    // Регистрация политики авторизации
    Gate::define('delete-post', function ($user, $post) {
      return $user->is($post->author);
    });
  }
}
```

## Создание поставщика служб

Мы создадим поставщика служб для нашего пакета, который будет содержать конкретную информацию о ядре нашего пакета. Пакет может использовать файл конфигурации, возможно, некоторые шаблоны, маршруты, контроллеры, миграции баз данных, фабрики моделей, консольные команды и т.д. Поставщик служб должен **зарегистрировать** их. Мы обсудим каждую позицию в следующих главах.

Поскольку мы подключили Orchestra Testbench, мы можем расширить `Illuminate\Support\ServiceProvider` и создать собственного поставщика службы в каталоге `src/`:

```php
// 'src/BlogPackageServiceProvider.php'
<?php

namespace JohnDoe\BlogPackage;

use Illuminate\Support\ServiceProvider;

class BlogPackageServiceProvider extends ServiceProvider
{
  public function register()
  {
    //
  }

  public function boot()
  {
    //
  }
}
```

## Автозагрузка

Чтобы автоматически зарегистрировать поставщика в проекте Laravel с помощью автоматического обнаружения пакетов Laravel, мы добавляем нашего поставщика служб к ключу `extra` -> `laravel` -> `providers` в файле `composer.json` нашего пакета:

```json
{
  ...,

  "autoload": { ... },

  "extra": {
      "laravel": {
          "providers": [
              "JohnDoe\\BlogPackage\\BlogPackageServiceProvider"
          ]
      }
  }
}
```

Теперь, когда кто-то затребует наш пакет, то будет загружен поставщик служб, и все, что мы зарегистрировали, будет доступно в приложении. <!--Теперь посмотрим, что мы можем зарегистрировать в этом поставщике служб.-->
