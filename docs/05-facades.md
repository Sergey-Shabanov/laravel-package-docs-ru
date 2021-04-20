<!-- ---
title: 'Facades'
description: 'Facades can provide end-users of your package with an easy-to-use (and understand) API for interaction with the functions (features) within your package. This section explains how to create your facades for your package.'
tags: ['Facades', 'API']
image: 'https://www.laravelpackage.com/assets/pages/laravelpackage.jpeg'
date: 2019-09-17
--- -->

# Разработка пакетов для Laravel · Фасады

The word 'facade' refers to a "superficial appearance or illusion of something," according to [Dictionary.com](https://www.dictionary.com/browse/facade). In architecture, the term refers to the front of a building.

A facade in Laravel is a class that redirects **static** method calls to the **dynamic** methods of an underlying class. A facade's goal is to provide a memorable and expressive syntax to access an underlying class's functionality.

An example of a fluent API using a facade:

```php
MessageFactory::sentBy($user)
    ->withTopic('Example message')
    ->withMessage($body)
    ->withReply($replyByFrank)
    ->create();
```

## How a Facade Works

To learn more about facades and how they work, refer to the excellent [Laravel documentation](https://laravel.com/docs/facades#how-facades-work).

Practically, it boils down to calling static methods on a Facade, which are "proxied" (redirected) to the non-static methods of an underlying class you have specified. This means that you're not *actually* using static methods. An example is discussed below, using a `Calculator` class as an example.

## Creating a Facade

Let’s assume that we provide a `Calculator` class as part of our package and want to make this class available as a facade.

First create a `Calculator.php` file in the `src/` directory. To keep things simple, the calculator provides an `add()`, `subtract()` and `clear()` method. All methods return the object itself allowing for a fluent API (chaining the method calls, like: `->add()->subtract()->subtract()->result()`).

```php
// 'src/Calculator.php'
<?php

namespace JohnDoe\BlogPackage;

class Calculator
{
    private $result;

    public function __construct()
    {
        $this->result = 0;
    }

    public function add(int $value)
    {
        $this->result += $value;

        return $this;
    }

    public function subtract(int $value)
    {
        $this->result -= $value;

        return $this;
    }

    public function clear()
    {
      $this->result = 0;

      return $this;
    }

    public function getResult()
    {
        return $this->result;
    }
}
```

In addition to this class, we’ll create the facade in a new `src/Facades` folder:

```php
// 'src/Facades/Calculator.php'
<?php

namespace JohnDoe\BlogPackage\Facades;

use Illuminate\Support\Facades\Facade;

class Calculator extends Facade
{
    protected static function getFacadeAccessor()
    {
        return 'calculator';
    }
}
```

Finally, we register the binding in the service container in our service provider:

```php
// BlogPackageServiceProvider.php
namespace JohnDoe\BlogPackage;

public function register()
{
  $this->app->bind('calculator', function($app) {
      return new Calculator();
  });
}
```

The end user can now use the `Calculator` facade after importing it from the appropriate namespace: `use JohnDoe\BlogPackage\Facades\Calculator;`. However, Laravel allows us to register an alias that can register a facade in the root namespace. We can define our alias under an “alias” key below the “providers” in the `composer.json` file:

```json
"extra": {
    "laravel": {
        "providers": [
            "JohnDoe\\BlogPackage\\BlogPackageServiceProvider"
        ],
        "aliases": {
            "Calculator": "JohnDoe\\BlogPackage\\Facades\\Calculator"
        }
    }
}
```

Our facade now no longer requires an import and can be used in projects from the root namespace:

```php
// Usage of the example Calculator facade
Calculator::add(5)->subtract(3)->getResult(); // 2
```
