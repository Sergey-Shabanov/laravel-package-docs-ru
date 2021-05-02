<!-- ---
title: "Routing, Views and Controllers"
description: "Expose (custom) routes in your package, which call a controller action and render views provided by the package. This chapter will additionally cover testing of routes, controllers, and views."
tags:
  [
    "Routing",
    "Controllers",
    "Views",
    "RESTful",
    "Testing Routing",
    "Testing Controllers",
    "Testing Views",
  ]
image: "https://www.laravelpackage.com/assets/pages/laravelpackage.jpeg"
date: 2019-09-17
--- -->

# Разработка пакетов для Laravel · Маршруты, контроллеры и шаблоны

Иногда вы хотите предоставить дополнительные маршруты конечному пользователю вашего пакета.

Поскольку мы предлагаем модель `Post`, давайте добавим несколько маршрутов **RESTful**. Для простоты мы просто реализуем 3 маршрута:

- показать все посты ('index')
- показать конкретный пост ('show')
- сохранить новый пост ('store')

## Контроллеры

### Создание базового контроллера

Мы хотим создать `PostController`.

Чтобы использовать некоторые трейты, предлагаемые контроллерам Laravel, мы сначала создадим наш собственный базовый контроллер, который будет содержать эти трейты, в каталоге `src/Http/Controllers`, напоминающем структуру папок Laravel, с именем `Controller.php`:

```php
// 'src/Http/Controllers/Controller.php'
<?php

namespace JohnDoe\BlogPackage\Http\Controllers;

use Illuminate\Foundation\Bus\DispatchesJobs;
use Illuminate\Routing\Controller as BaseController;
use Illuminate\Foundation\Validation\ValidatesRequests;
use Illuminate\Foundation\Auth\Access\AuthorizesRequests;

class Controller extends BaseController
{
    use AuthorizesRequests, DispatchesJobs, ValidatesRequests;
}
```

### Создание контроллера, расширяющего базовый контроллер

Теперь давайте создадим `PostController` в каталоге `src/Http/Controllers`, начав его наполнение с метода `store`:

```php
// 'src/Http/Controllers/PostController'
<?php

namespace JohnDoe\BlogPackage\Http\Controllers;

class PostController extends Controller
{
    public function index()
    {
        //
    }

    public function show()
    {
        //
    }

    public function store()
    {
        // Предположим, нам нужно пройти аутентификацию,
        // чтобы создать новый пост.
        if (! auth()->check()) {
            abort (403, 'Only authenticated users can create new posts.');
        }

        request()->validate([
            'title' => 'required',
            'body'  => 'required',
        ]);

        // Предположим, что аутентифицированный пользователь является автором сообщения.
        $author = auth()->user();

        $post = $author->posts()->create([
            'title'     => request('title'),
            'body'      => request('body'),
        ]);

        return redirect(route('posts.show', $post));
    }
}
```

## Маршруты

### Определение маршрутов

Теперь, когда у нас есть контроллер, создайте новый каталог `routes/` в корне нашего пакета и добавьте файл `web.php`, содержащий три маршрута RESTful, о которых мы упоминали выше.

```php
// 'routes/web.php'
<?php

use Illuminate\Support\Facades\Route;
use JohnDoe\BlogPackage\Http\Controllers\PostController;

Route::get('/posts', [PostController::class, 'index'])->name('posts.index');
Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');
Route::post('/posts', [PostController::class, 'store'])->name('posts.store');
```

### Регистрация маршрутов в поставщике служб

Прежде чем мы сможем использовать эти маршруты, нам необходимо зарегистрировать их в методе `boot()` нашего поставщика служб:

```php
// 'BlogPackageServiceProvider.php'
public function boot()
{
  // Остальной код ...
  $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
}
```

### Конфигурируемые префикс маршрута и посредники

Вы можете разрешить пользователям определять префикс и посредники для маршрутов, предоставляемых вашим пакетом. Вместо того, чтобы регистрировать маршруты непосредственно в методе `boot()`, мы зарегистрируем маршруты, используя `Route::group`, передав динамическую конфигурацию: префикс и посредники. Не забудьте импортировать соответствующий фасад `Route`.

В следующих примерах используется имя конфигурационного файла `blogpackage`. Не забудьте заменить на имя конфигурационного файла вашего пакета.

```php
// 'BlogPackageServiceProvider.php'
use Illuminate\Support\Facades\Route;

public function boot()
{
  // Остальной код ...
  $this->registerRoutes();
}

protected function registerRoutes()
{
    Route::group($this->routeConfiguration(), function () {
        $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
    });
}

protected function routeConfiguration()
{
    return [
        'prefix' => config('blogpackage.prefix'),
        'middleware' => config('blogpackage.middleware'),
    ];
}
```

Укажите префикс маршрута по умолчанию и посредники в файле пакета `config.php`:

```php
'prefix' => 'blogger',
'middleware' => ['web'], // вероятно, вы захотите использовать здесь `web`
```

В приведенной выше конфигурации по умолчанию все маршруты, определенные в `routes.web`, должны иметь префикс `/blogger`. Таким образом вы избежите конфликтов с потенциально существующими маршрутами.

## Шаблоны

Методы `index` и `show` в `PostController` должны возвращать представления.

### Создание файлов шаблонов Blade

Создайте новый каталог `resources/` в корне нашего пакета. В этом каталоге создайте подпапку с именем `views`. В папке `views` мы создадим подпапку `posts`, в которой мы создадим два «экстремально» простых шаблона.

1. `resources/views/posts/index.blade.php`:

   ```
   <h1>Showing all Posts</h1>

   @forelse ($posts as $post)
       <li>{{ $post->title }}</li>
   @empty
       <p> 'No posts yet' </p>
   @endforelse
   ```

2. `resources/views/posts/show.blade.php`:

   ```
   <h1>{{ $post->title }}</h1>

   <p> {{ $post->body }}</p>
   ```

**Примечание**: эти шаблоны могут расширить базовый / главный макет в реальном сценарии.

### Регистрация шаблонов в поставщике служб

Теперь, когда у нас есть несколько шаблонов, нам нужно зарегистрировать, что мы хотим загружать любые шаблоны из нашего каталога `resources/views` в методе `boot()` нашего поставщика служб. **Важно**: укажите «ключ» в качестве второго аргумента для `loadViewsFrom()`, так как вам нужно будет указать этот ключ при возврате шаблона из контроллера (см. следующий раздел).

```php
// 'BlogPackageServiceProvider.php'
public function boot()
{
  // Остальной код ...
  $this->loadViewsFrom(__DIR__.'/../resources/views', 'blogpackage');
}
```

### Возвращение шаблона из контроллера

Теперь мы можем вернуть созданный нами шаблон из `PostController`. Не забудьте импортировать нашу модель `Post`.

Обратите внимание на префикс `blogpackage::`, который соответствует префиксу, зарегистрированному в нашем поставщике служб.

```php
// 'src/Http/Controllers/PostController.php'
use JohnDoe\BlogPackage\Models\Post;

public function index()
{
    $posts = Post::all();

    return view('blogpackage::posts.index', compact('posts'));
}

public function show()
{
    $post = Post::findOrFail(request('post'));

    return view('blogpackage::posts.show', compact('post'));
}
```

### Редактирование шаблонов пользователями пакета

Скорее всего, вы захотите разрешить пользователям вашего пакета _изменять_ шаблоны. Подобно миграции базы данных, шаблоны могут быть **опубликованы**, если мы зарегистрируем их для экспорта в методе `boot()` нашего поставщика служб, используя ключ `views` метода `publishes()`:

```php
// 'BlogPackageServiceProvider.php'
if ($this->app->runningInConsole()) {
  // Опубликовать шаблоны
  $this->publishes([
    __DIR__.'/../resources/views' => resource_path('views/vendor/blogpackage'),
  ], 'views');

}
```

Затем шаблоны могут быть экспортированы пользователями нашего пакета с помощью команды:

```
php artisan vendor:publish --provider="JohnDoe\BlogPackage\BlogPackageServiceProvider" --tag="views"
```

## View Components

Since Laravel 8, it is possible to generate Blade components using `php artisan make:component MyComponent` which generates a base `MyComponent` class and a Blade `my-component.blade.php` file, which receives all public properties as defined in the `MyComponent` class. These components can then be reused and included in any view using the component syntax: `<x-my-component>` and closing `</x-my-component>` (or the self-closing form). To learn more about Blade components, make sure to check out the Laravel documentation.

In addition to generating Blade components using the artisan command, it is also possible to create a `my-component.blade.php` component without class. These are called anonymous components and are placed in the `views/components` directory by convention.

This section will cover how to provide these type of Blade components in your package.

### Class Based Components

If you want to offer class based View Components in your package, first create a new `View/Components` directory in the `src` folder. Add a new class, for example `Alert.php`.

```php
// 'src/View/Components/Alert.php'
<?php

namespace JohnDoe\BlogPackage\View\Components;

use Illuminate\View\Component;

class Alert extends Component
{
    public $message;

    public function __construct($message)
    {
        $this->message = $message;
    }

    public function render()
    {
        return view('blogpackage::components.alert');
    }
}
```

Next, create a new `views/components` directory in the `resources` folder. Add a new Blade component `alert.blade.php`:

```html
<div>
  <p>This is an Alert</p>

  <p>{{ $message }}</p>
</div>
```

Next, register the component in the Service Provider by the class and provide a prefix for the components. In our example, using 'blogpackage', the alert component will become available as `<x-blogpackage-alert />`.

```php
// 'BlogPackageServiceProvider.php'
<?php

use JohnDoe\BlogPackage\View\Components\Alert;

public function boot()
{
  // ... other things
  $this->loadViewComponentsAs('blogpackage', [
    Alert::class,
  ]);
}
```

### Anonymous View Components

If your package provides anonymous components, it suffices to add the `my-component.blade.php` Blade component to `resources/views/components` directory, given that you have specified the `loadViewsFrom` directory in your Service Provider as "resources/views". If you don't already, add the `loadViewsFrom` method to your Service Provider:

```php
// 'BlogPackageServiceProvider.php'
public function boot()
{
  // ... other things
  $this->loadViewsFrom(__DIR__.'/../resources/views', 'blogpackage');
}
```

Components (in the `resources/views/components` folder) can now be referenced prefixed by the defined namespace above ("blogpackage"):

```
  <x-blogpackage::alert />
```

### Customizable View Components

In order to let the end user of our package modify the provided Blade component(s), we first need to register the publishables into our Service Provider:

```php
// 'BlogPackageServiceProvider.php'
if ($this->app->runningInConsole()) {
  // Publish view components
  $this->publishes([
      __DIR__.'/../src/View/Components/' => app_path('View/Components'),
      __DIR__.'/../resources/views/components/' => resource_path('views/components'),
  ], 'view-components');
}
```

Now, it is possible to publish both files (class and Blade component) using:

```
php artisan vendor:publish --provider="JohnDoe\BlogPackage\BlogPackageServiceProvider" --tag="view-components"
```

Be aware that the end user needs to update the namespaces of the published component class and update the `render()` method to reference the Blade components of the Laravel application directly, instead of referencing the package namespace. Additionally, the Blade component no longer has to be namespaced since it was published to the Laravel application itself.

## Веб-активы

Скорее всего, вы захотите также включить файлы CSS и Javascript при добавлении шаблонов в свой пакет.

### Создание директории `assets`

Если вы хотите использовать таблицу стилей CSS или включить файл Javascript в свои шаблоны, то создайте каталог `resources/assets`. Поскольку мы можем использовать как CSS, так и Javascript, то давайте создадим **две подпапки**: `css` и `js` для хранения этих файлов, соответственно. По соглашению основной файл Javascript называется `app.js`, а основная таблица стилей – `app.css`.

### Редактирование веб-активов пользователями пакета

Как и в случае с шаблонами, мы можем позволить нашим пользователям по желанию изменять веб-активы. Во-первых, мы определим, куда мы будем экспортировать активы в методе `boot()` нашего поставщика служб под ключом `assets` в каталог `public/blogpackage` приложения Laravel конечного пользователя:

```php
// 'BlogPackageServiceProvider.php'
if ($this->app->runningInConsole()) {
  // Опубликовать веб-активы
  $this->publishes([
    __DIR__.'/../resources/assets' => public_path('blogpackage'),
  ], 'assets');

}
```

Затем веб-активы могут быть экспортированы пользователями нашего пакета с помощью команды:

```
php artisan vendor:publish --provider="JohnDoe\BlogPackage\BlogPackageServiceProvider" --tag="assets"
```

### Ссылка на веб-активы

Мы можем сослаться на таблицу стилей и файл Javascript в шаблонах следующим образом:

```html
<script src="{{ asset('blogpackage/js/app.js') }}"></script>
<link href="{{ asset('blogpackage/css/app.css') }}" rel="stylesheet" />
```

## Тестирование маршрутов

Давайте проверим, действительно ли мы можем создать пост, показать пост и показать все посты с предоставленными нами маршрутами, шаблонами и контроллерами.

### Функциональный тест

Создайте новый функциональный тест с именем `CreatePostTest.php` в каталоге `tests/Feature` и добавьте следующие утверждения, чтобы убедиться, что аутентифицированные пользователи действительно могут создавать новые посты:

```php
// 'tests/Feature/CreatePostTest.php'
<?php

namespace JohnDoe\BlogPackage\Tests\Feature;

use Illuminate\Foundation\Testing\RefreshDatabase;
use JohnDoe\BlogPackage\Models\Post;
use JohnDoe\BlogPackage\Tests\TestCase;
use JohnDoe\BlogPackage\Tests\User;

class CreatePostTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    function authenticated_users_can_create_a_post()
    {
        // Чтобы убедиться в отсутствии постов.
        $this->assertCount(0, Post::all());

        $author = User::factory()->create();

        $response = $this->actingAs($author)->post(route('posts.store'), [
            'title' => 'My first fake title',
            'body'  => 'My first fake body',
        ]);

        $this->assertCount(1, Post::all());

        tap(Post::first(), function ($post) use ($response, $author) {
            $this->assertEquals('My first fake title', $post->title);
            $this->assertEquals('My first fake body', $post->body);
            $this->assertTrue($post->author->is($author));
            $response->assertRedirect(route('posts.show', $post));
        });
    }
}
```

Кроме того, мы может убедиться, что при создании нового поста нам обязательно требуются атрибуты `title` и `body`:

```php
// 'tests/Feature/CreatePostTest.php'
/** @test */
function a_post_requires_a_title_and_a_body()
{
    $author = User::factory()->create();

    $this->actingAs($author)->post(route('posts.store'), [
        'title' => '',
        'body'  => 'Some valid body',
    ])->assertSessionHasErrors('title');

    $this->actingAs($author)->post(route('posts.store'), [
        'title' => 'Some valid title',
        'body'  => '',
    ])->assertSessionHasErrors('body');
}
```

Затем давайте проверим, что неаутентифицированные пользователи или «гости» не могут создавать новые посты:

```php
// 'tests/Feature/CreatePostTest.php'
/** @test */
function guests_can_not_create_posts()
{
    // Мы начинаем с неаутентифицированного состояния
    $this->assertFalse(auth()->check());

    $this->post(route('posts.store'), [
       'title' => 'A valid title',
       'body'  => 'A valid body',
    ])->assertForbidden();
}
```

Наконец, давайте проверим, что маршрут `index` показывает все посты, а маршрут `show` показывает конкретный пост:

```php
// 'tests/Feature/CreatePostTest.php'
/** @test */
function all_posts_are_shown_via_the_index_route()
{
    // Учитывая, что у нас есть тройка сообщений
    Post::factory()->create([
        'title' => 'Post number 1'
    ]);
    Post::factory()->create([
        'title' => 'Post number 2'
    ]);
    Post::factory()->create([
        'title' => 'Post number 3'
    ]);

    // Мы ожидаем, что они все появятся
    // с их заголовком в индексном маршруте
    $this->get(route('posts.index'))
        ->assertSee('Post number 1')
        ->assertSee('Post number 2')
        ->assertSee('Post number 3')
        ->assertDontSee('Post number 4');
}

/** @test */
function a_single_post_is_shown_via_the_show_route()
{
    $post = Post::factory()->create([
        'title' => 'The single post title',
        'body'  => 'The single post body',
    ]);

    $this->get(route('posts.show', $post))
        ->assertSee('The single post title')
        ->assertSee('The single post body');
}
```

> Совет: всякий раз, когда вы получаете загадочные сообщения об ошибках из ваших тестов, может быть полезно отключить «изящную» обработку исключений, чтобы лучше понять причину ошибки. Вы можете сделать это, объявив `$this->withoutExceptionHandling();` в начале вашего теста.
