<!-- ---
title: 'Middleware'
description: 'Explore the different types of Middleware and how to make use of them within a Laravel package. Additionally, writing tests for the Middleware will be explained.'
tags: ['Middleware', 'Before Middlware', 'After Middleware', 'Route Middleware', 'Middleware Groups', 'Global Middleware', 'Testing Middleware']
image: 'https://www.laravelpackage.com/assets/pages/laravelpackage.jpeg'
date: 2019-09-17
--- -->

# Разработка пакетов для Laravel · Посредники

Если мы посмотрим на входящий HTTP-запрос, этот запрос обрабатывается файлом `index.php` Laravel и отправляется через серию конвейеров. К ним относятся серия посредников «до», каждый из которых будет действовать в соответствии с входящим запросом, прежде чем он в конечном итоге достигнет ядра приложения. Ответ подготавливается из ядра приложения и пост-модифицируется перед возвратом всеми зарегистрированными посредниками серии «после».

Вот почему посредники отлично подходит для аутентификации, проверки токенов или применения любой другой проверки. Laravel также использует посредников для удаления пустых символов из строк и шифрования файлов Cookies.

## Создание посредника

Существуют два типа посредников: 1) взаимодействующие с запросом до возврата ответа; 2) взаимодействующие с ответом перед его возвратом.

Прежде чем обсуждать оба типа посредников, создайте новую папку `Middleware` в каталоге `src/Http` пакета.

## Посредник запроса

Посредник _запроса_ выполняет действие над запросом, а затем вызывает следующий посредник в очереди. Как правило, посредник _запроса_ имеет следующий вид:

```php
<?php

namespace App\Http\Middleware;

use Closure;

class BeforeMiddleware
{
    public function handle($request, Closure $next)
    {
        // Выполнить действие ...

        return $next($request);
    }
}
```

В качестве иллюстрации посредника запроса давайте добавим посредника, который преобразует первый символ входящего параметра `title` в верхний регистр, если тот присутствует в запросе, что было бы глупо в реальном приложении.

Добавьте файл с именем `CapitalizeTitle.php`, который содержит метод `handle()`, принимающий как текущий запрос, так и действие `$next`:

```php
// 'src/Http/Middleware/CapitalizeTitle.php'
<?php

namespace JohnDoe\BlogPackage\Http\Middleware;

use Closure;

class CapitalizeTitle
{
    public function handle($request, Closure $next)
    {
        if ($request->has('title')) {
            $request->merge([
                'title' => ucfirst($request->title)
            ]);
        }

        return $next($request);
    }
}
```

## Тестирование посредника запроса

Хотя мы еще не _зарегистрировали_ посредника, да и в приложении использоваться он не будет, мы хотим убедиться, что метод `handle()` отражает ожидаемое поведение.

Добавьте новый модульный тест `CapitalizeTitleMiddlewareTest.php` в каталог `tests/Unit`. В этом тесте мы будем утверждать, что параметр `title` запроса будет содержать строку с заглавной буквы после того, как посредник выполнит свой метод `handle()`:

```php
// 'tests/Unit/CapitalizeMiddlewareTest.php'
<?php

namespace JohnDoe\BlogPackage\Tests\Unit;

use Illuminate\Http\Request;
use JohnDoe\BlogPackage\Http\Middleware\CapitalizeTitle;
use JohnDoe\BlogPackage\Tests\TestCase;

class CapitalizeTitleMiddlewareTest extends TestCase
{
    /** @test */
    function it_capitalizes_the_request_title()
    {
        // Учитывая, что у нас есть запрос
        $request = new Request();

        // с параметром `title`, состоящий из строчных букв
        $request->merge(['title' => 'some title']);

        // когда мы передаем запрос этому посреднику,
        // `title` должен начинаться с заглавной буквы
        (new CapitalizeTitle())->handle($request, function ($request) {
            $this->assertEquals('Some title', $request->title);
        });
    }
}
```

## Посредник ответа

Посредник _ответа_ взаимодействует с ответом, возвращенным по цепочке после прохождения через других посредников. Затем он изменяет и возвращает ответ. Обычно он имеет следующий вид:

```php
<?php

namespace App\Http\Middleware;

use Closure;

class AfterMiddleware
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        // Выполнить действие ...

        return $response;
    }
}
```

## Тестирование посредника ответа

Подобно *посреднику запроса*, мы можем провести модульное тестирование *посредника ответа*, который взаимодействует с ответом текущего запроса, изменяя этот запрос, прежде чем он будет передан следующему посреднику. Предположим, что у нас есть посредник `InjectHelloWorld`, который зачем-то вставляет строку `Hello World` в каждый ответ, следующий тест должен стать подтверждением ожидаемого поведения:

```php
// 'tests/Unit/InjectHelloWorldMiddlewareTest.php'
<?php

namespace JohnDoe\BlogPackage\Tests\Unit;

use Illuminate\Http\Request;
use JohnDoe\BlogPackage\Http\Middleware\InjectHelloWorld;
use JohnDoe\BlogPackage\Tests\TestCase;

class InjectHelloWorldMiddlewareTest extends TestCase
{
    /** @test */
    function it_checks_for_a_hello_word_in_response()
    {
        // Учитывая, что у нас есть запрос
        $request = new Request();

        // когда мы передаем запрос этому посреднику,
        // ответ должен содержать строку «Hello World»
        $response = (new InjectHelloWorld())->handle($request, function ($request) { });

        $this->assertStringContainsString('Hello World', $response);
    }
}
```

Теперь, когда мы знаем, что метод `handle()` выполняет свою работу правильно, давайте рассмотрим два варианта регистрации посредников: **глобально** и **для конкретного маршрута**.

## Глобальный посредник

Глобальный посредник, как следует из названия, применяется глобально. Каждый запрос будет проходить через этот посредник.

Если мы хотим, чтобы наш пример с заглавной буквой применялся глобально, мы можем добавить этот посредник в `Http\Kernel` через поставщика служб нашего пакета. Обязательно импортируйте контракт `Http\Kernel`, а не контракт `Console\Kernel`:

```php
// 'BlogPackageServiceProvider.php'
use Illuminate\Contracts\Http\Kernel;
use JohnDoe\BlogPackage\Http\Middleware\CapitalizeTitle;

public function boot()
{
  // Остальной код ...

  $kernel = $this->app->make(Kernel::class);
  $kernel->pushMiddleware(CapitalizeTitle::class);
}
```

Это добавит наш посредник в массив глобально зарегистрированных посредников приложения.

## Посредник маршрута

В нашем случае вы можете возразить, что у нас, скорее всего, нет параметра `title` для каждого запроса. Возможно даже только по запросам, связанным с созданием / обновлением постов. Вдобавок к этому мы, вероятно, когда-нибудь захотим применить этот посредник только к запросам, связанным с постами в нашем блоге.

Однако посредник в нашем примере изменит все запросы, у которых есть атрибут `title`. Вероятно, это нежелательно. Решение состоит в том, чтобы сделать посредник специфичным для маршрута.

Мы можем зарегистрировать псевдоним для этого посредника с помощью извлеченного класса `Router` в методе `boot()` нашего поставщика служб.

Вот как можно зарегистрировать псевдоним `capitalize` для этого посредника:

```php
// 'BlogPackageServiceProvider.php'
use Illuminate\Routing\Router;
use JohnDoe\BlogPackage\Http\Middleware\CapitalizeTitle;

public function boot()
{
  // Остальной код ...

  $router = $this->app->make(Router::class);
  $router->aliasMiddleware('capitalize', CapitalizeTitle::class);
}
```

Мы можем применить этот посредник к нашему контроллеру, указав зависимость в конструкторе:

```php
// 'src/Http/Controllers/PostController.php'
class PostController extends Controller
{
    public function __construct()
    {
        $this->middleware('capitalize');
    }

    // Другие методы будут использовать этот посредник ...
}
```

### Группы посредников

Кроме того, мы можем добавить наш посредник в определенные группы, такие как `web` или `api`, чтобы убедиться, что наш посредник применяется для каждого маршрута, который принадлежит этим группам.

Чтобы определить посредника в определенную группу, вызовите метод `push` маршрутизатора, например, `web`:

```php
// 'BlogPackageServiceProvider.php'
use Illuminate\Routing\Router;
use JohnDoe\BlogPackage\Http\Middleware\CapitalizeTitle;

public function boot()
{
  // Остальной код ...

  $router = $this->app->make(Router::class);
  $router->pushMiddlewareToGroup('web', CapitalizeTitle::class);
}
```

Группы посредников маршрутов приложения Laravel расположены в классе `App\Http\Kernel`. Применяя этот подход, вы должны быть уверены, что у пользователей этого пакета есть указанная группа посредников, определенная в их приложении.

## Функциональное тестирование посредника

Независимо от того, зарегистрировали ли мы посредник глобально или локально, мы можем проверить, применяется ли посредник при выполнении запроса.

Добавьте новый тест `CreatePostTest` в каталог `Feature`. В данном тесте мы будем предполагать, что наш заголовок, состоящий из строчных букв, будет написан с заглавной буквы после того, как запрос будет выполнен.

```php
// 'tests/Feature/CreatePostTest.php'
/** @test */
function creating_a_post_will_capitalize_the_title()
{
    $author = User::factory()->create();

    $this->actingAs($author)->post(route('posts.store'), [
        'title' => 'some title that was not capitalized',
        'body' => 'A valid body',
    ]);

    $post = Post::first();

    // Префикс `New: ` был добавлен нашим слушателем событий.
    $this->assertEquals('New: Some title that was not capitalized', $post->title);
}
```

Когда тесты вернет зеленый, тогда будем считать, что мы рассмотрели добавление посредника в ваш пакет.
