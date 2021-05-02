<!-- ---
title: "Models and Migrations"
description: "Some packages need to offer a Laravel Model. This section explains how to allow for this and include your own database migrations. Additionally, the section will cover testing the models and migrations."
tags: ["Models", "Migrations", "Testing Models", "Unit Test"]
image: "https://www.laravelpackage.com/assets/pages/laravelpackage.jpeg"
date: 2019-09-17
--- -->

# Разработка пакетов для Laravel · Модели и миграции

Есть сценарии, в которых вам нужно будет добавить в ваш пакет одну или несколько моделей Eloquent. Например, когда вы разрабатываете связанный с блогом пакет, содержащий модель `Post`.

В этой главе будет рассказано, как добавить модели Eloquent в ваш пакет, включая миграции, тесты и как, возможно, добавить отношения к модели `App\User`, которая уже содержится в приложениях Laravel.

## Модели

Модели нашего пакета не отличаются от моделей, которые мы использовали бы в стандартном приложении Laravel. Поскольку мы используем **Orchestra Testbench**, то мы можем создать модель, расширяющую модель Laravel Eloquent, и сохранить ее в каталоге `src/Models`:

```php
// 'src/Models/Post.php'
<?php

namespace JohnDoe\BlogPackage\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
  use HasFactory;

  // Отключить защиту от массового присвоения Laravel.
  protected $guarded = [];
}
```

Существует несколько способов генерации моделей вместе с автоматической миграцией. Простой подход – использовать обычное приложение Laravel, а затем переместить файлы, сгенерированные с помощью Artisan, в свой пакет, а затем изменить их пространства имен.

Если вы ищете способы автоматизации создания каркасов в вашем пакете, то вы можете установить один из следующих инструментов в качестве зависимости `dev` в вашем пакете и использовать команду интерфейса командной строки для генерации каркасов.

- [Laravel Package Tools](https://github.com/beyondcode/laravel-package-tools)
- [Laravel Packer](https://github.com/bitfumes/laravel-packer)
- [Laravel Package Maker](https://github.com/naoray/laravel-package-maker)

## Миграции

В приложении Laravel миграции находятся в каталоге `database/migrations`. В нашем пакете мы имитируем эту файловую структуру. Следовательно, миграции базы данных будут находиться не в каталоге `src/`, а в своем собственном каталоге `database/migrations`. Корневой каталог нашего пакета теперь содержит как минимум два каталога: `src/` и `database/`.

После того, как вы сгенерировали миграцию, скопируйте ее из своего «фиктивного» приложения Laravel в каталог пакета `database/migrations`.

```php
// 'database/migrations/2018_08_08_100000_create_posts_table.php'
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreatePostsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('posts');
    }
}
```

С этого момента есть два возможных подхода к представлению конечному пользователю наших миграций. Мы можем либо опубликовать определенные миграции (метод 1), либо автоматически загрузить все миграции из нашего пакета (метод 2).

### Метод первый: публикация миграций

В этом подходе мы регистрируем, что наш пакет «публикует» свои миграции. Мы можем сделать это следующим образом в методе `boot()` поставщика служб нашего пакета, используя метод `publishes()`, который принимает два аргумента:

1. массив путей к файлам `["путь исходника" => "путь назначения"]`

2. тег, присваиваемый группе связанных публикуемых ресурсов.

В этом подходе обычно используется «заготовка» миграции. Эта заготовка экспортируется в реальную миграцию, когда пользователь нашего пакета публикует миграции. Поэтому переименуйте миграции, чтобы удалить отметку времени и добавить расширение `.stub`. В нашем примере миграции это приведет к именованию: `create_posts_table.php.stub`.

Затем мы можем реализовать экспорт миграции следующим образом:

```php
class BlogPackageServiceProvider extends ServiceProvider
{
  public function boot()
  {
    if ($this->app->runningInConsole()) {
      // Экспорт миграции.
      if (! class_exists('CreatePostsTable')) {
        $this->publishes([
          __DIR__ . '/../database/migrations/create_posts_table.php.stub' => database_path('migrations/' . date('Y_m_d_His', time()) . '_create_posts_table.php'),
          // Здесь вы можете добавить любое количество миграций.
        ], 'migrations');
      }
    }
  }
}
```

В приведенном выше коде мы сначала проверяем, запущено ли приложение в режиме консоли. Затем мы проверяем, публиковал ли пользователь миграции. Если нет, то мы опубликуем миграцию `create_posts_table` в каталоге `migrations` каталога `database` с префиксом текущей даты и времени.

Миграции этого пакета теперь можно публиковать, используя тег `migrations`:

```bash
php artisan vendor:publish --provider="JohnDoe\BlogPackage\BlogPackageServiceProvider" --tag="migrations"
```

### Метод второй: автоматическая загрузка миграций

Хотя описанный выше метод дает полный контроль над тем, какие миграции публиковать, Laravel предлагает альтернативный подход, использующий метод `loadMigrationsFrom` ([см. документацию](https://github.com/russsiq/laravel-docs-8.x-ru/blob/main/docs/packages.md#migrations)). Если указать каталог миграции в поставщике служб пакета, то все миграции будут запущены, когда конечный пользователь выполнит `php artisan migrate` из своего приложения Laravel.

```php
class BlogPackageServiceProvider extends ServiceProvider
{
  public function boot()
  {
    $this->loadMigrationsFrom(__DIR__ . '/../database/migrations');
  }
}
```

Убедитесь, что вы указали правильную временную метку для своих миграций, иначе Laravel не сможет их обработать. Например: `2018_08_08_100000_example_migration.php`. При таком подходе нельзя использовать заготовку как это определено в [первом методе](#метод-первый-публикация-миграций).

## Тестирование моделей и миграций

Создавая пример теста, мы будем следовать некоторым основам разработки через тестирование (TDD). Независимо от того, практикуете ли вы TDD в своем привычном рабочем процессе, объяснение шагов здесь помогает выявить возможные проблемы, с которыми вы можете столкнуться на этом пути, что упрощает устранение неполадок. Давайте начнем.

### Написание модульных тестов

Теперь, когда мы настроили **PHPunit**, давайте создадим модульный тест для нашей модели `Post` в каталоге `tests/Unit` с именем `PostTest.php`. Давайте напишем тест, который проверяет наличие заголовка у поста:

```php
// 'tests/Unit/PostTest.php'
<?php

namespace JohnDoe\BlogPackage\Tests\Unit;

use Illuminate\Foundation\Testing\RefreshDatabase;
use JohnDoe\BlogPackage\Tests\TestCase;
use JohnDoe\BlogPackage\Models\Post;

class PostTest extends TestCase
{
  use RefreshDatabase;

  /** @test */
  function a_post_has_a_title()
  {
    $post = Post::factory()->create(['title' => 'Fake Title']);
    $this->assertEquals('Fake Title', $post->title);
  }
}
```

> **Примечание**: мы используем трейт `RefreshDatabase`, чтобы быть уверенным, что мы начинаем с пустой базой данных перед каждым тестом.

### Запуск тестов

Мы можем запустить наш набор тестов, вызвав двоичный файл PHPUnit в нашем каталоге `vendor`, используя `./vendor/bin/phpunit`. Однако давайте добавим псевдоним `test` в нашем файле `composer.json` в ключе `scripts`:

```json
{
  ...,

  "autoload-dev": {},

  "scripts": {
    "test": "vendor/bin/phpunit",
    "test-f": "vendor/bin/phpunit --filter"
  }
}
```

Теперь мы можем выполнить команду `composer test`, чтобы запустить все наши тесты или `composer test-f` с именем тестового класса / метода, чтобы запустить только указанный класс / метод.

Выполнение команды `composer test-f a_post_has_a_title` приведет нас к следующей ошибке:

```
Error: Class 'Database\Factories\JohnDoe\BlogPackage\Models\PostFactory' not found
```

Вышеупомянутая ошибка говорит нам о необходимости создать фабрику модели `Post`.

### Создание фабрики модели

Давайте создадим `PostFactory` в каталоге `database/factories`:

```php
// 'database/factories/PostFactory.php'
<?php

namespace JohnDoe\BlogPackage\Database\Factories;

use JohnDoe\BlogPackage\Models\Post;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    protected $model = Post::class;

    public function definition()
    {
        return [
            //
        ];
    }
}

```

Как и в случае с каталогом `src`, для того, чтобы пользователи нашего пакета могли использовать наши фабрики моделей, нам нужно зарегистрировать каталог `database/factories` в пространстве имен в нашем файле `composer.json`:

```json
{
  ...,
  "autoload": {
    "psr-4": {
      "JohnDoe\\BlogPackage\\": "src",
      "JohnDoe\\BlogPackage\\Database\\Factories\\": "database/factories"
    }
  },
  ...
}
```

После изменений не забудьте выполнить `composer dump-autoload`.

### Конфигурирование фабрики модели

Повторный запуск наших тестов приведет к следующей ошибке:

```
Error: Class 'Database\Factories\JohnDoe\BlogPackage\Models\PostFactory' not found
```

Вышеупомянутая ошибка вызвана Laravel, который пытается найти класс `Model` для `PostFactory`, принимая пространства имен `App\Models` по умолчанию для обычного проекта.
Чтобы создать экземпляр требуемой модели из нашего пакета с помощью метода `Post::factory()`, необходимо добавить следующий метод в нашу модель `Post`:

```php
// 'src/Models/Post.php'

protected static function newFactory()
{
    return \JohnDoe\BlogPackage\Database\Factories\PostFactory::new();
}
```

Однако тесты все равно не будут пройдены, поскольку мы не создали таблицу `posts` в нашей базе данных SQLite. Нам нужно указать нашим тестам, чтобы они выполняли все миграции перед запуском тестов.

Давайте загрузим миграции в методе `getEnvironmentSetUp()` нашего `TestCase`:

```php
// 'tests/TestCase.php'

public function getEnvironmentSetUp($app)
{
  // Импортировать класс `CreatePostsTable` из миграции.
  include_once __DIR__ . '/../database/migrations/create_posts_table.php.stub';

  // Выполнить метод `up()` импортированного класса миграции.
  (new \CreatePostsTable)->up();
}
```

Теперь повторный запуск тестов приведет к ожидаемой ошибке: в таблице `posts` нет столбца `title`. Давайте исправим это миграции `create_posts_table.php.stub`:

```php
// 'database/migrations/create_posts_table.php.stub'
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->timestamps();
});

```

После запуска теста вы должны увидеть, что он был пройден.

### Добавление тестов для других столбцов

Добавим тесты для `body` и `author_id`:

```php
// 'tests/Unit/PostTest.php'
class PostTest extends TestCase
{
  use RefreshDatabase;

  /** @test */
  function a_post_has_a_title()
  {
    $post = Post::factory()->create(['title' => 'Fake Title']);
    $this->assertEquals('Fake Title', $post->title);
  }

  /** @test */
  function a_post_has_a_body()
  {
    $post = Post::factory()->create(['body' => 'Fake Body']);
    $this->assertEquals('Fake Body', $post->body);
  }

  /** @test */
  function a_post_has_an_author_id()
  {
    // Обратите внимание, что мы не предполагаем здесь отношений, просто у нас есть столбец для хранения идентификатора автора.
    $post = Post::factory()->create(['author_id' => 999]); // мы выбираем недопустимое значение для `author_id`, поэтому маловероятно, что возникнуть коллизии с другим `author_id` в наших тестах
    $this->assertEquals(999, $post->author_id);
  }
}
```

Вы можете продолжить работу над этим с помощью TDD самостоятельно: запустить тесты, определить следующий функционал, который необходимо реализовать, и снова запустить тесты.

В конце концов у вас получится фабрика модели и миграция как ниже:

```php
// 'database/factories/PostFactory.php'
<?php

namespace JohnDoe\BlogPackage\Database\Factories;

use JohnDoe\BlogPackage\Models\Post;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    protected $model = Post::class;

    public function definition()
    {
        return [
            'title'     => $this->faker->words(3, true),
            'body'      => $this->faker->paragraph,
            'author_id' => 999,
        ];
    }
}

```

На данный момент мы жестко запрограммировали `author_id`. В следующем разделе мы увидим, как можно урегулировать отношения с моделью `User`.

```php
// 'database/migrations/create_posts_table.php.stub'

Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('body');
    $table->unsignedBigInteger('author_id');
    $table->timestamps();
});
```

## Определение отношений с App\User

Теперь, когда у нас есть столбец `author_id` в нашей модели `Post`, давайте создадим связь между `Post` и `User`. Однако у нас есть проблема, поскольку нам нужна модель `User`, но эта модель включена по умолчанию во фреймворк Laravel ...

Мы не можем просто предоставить нашу собственную модель `User`, поскольку вы, вероятно, хотите, чтобы ваш конечный пользователь мог использовать модель `User` из своего приложения Laravel.

Ниже представлены два варианта создания отношения.

### Вариант первый: получение модели пользователя из конфигурации аутентификации

Если вы просто хотите создать связь между **аутентифицированными пользователями** и _например_ моделью `Post`, то самым простым вариантом является ссылка на модель, которая используется в файле `config/auth.php`. По умолчанию это модель Eloquent `App\Models\User`.

Если вы просто хотите назначить модель Eloquent, которая отвечает за аутентификацию, то создайте отношение `belongsToMany` в модели `Post` следующим образом:

```php
// Post model
class Post extends Model
{
  public function author()
  {
    return $this->belongsTo(config('auth.providers.users.model'));
  }
}
```

Однако что, если у пользователя нашего пакета есть модели `Admin` и `User` и автором поста может быть как модель `Admin`, так и `User`? В таких случаях вы можете выбрать полиморфные отношения.

### Вариант второй: использование полиморфных отношений

Вместо того, чтобы выбирать обычные отношения «один-ко-многим» (у пользователя может быть много постов, и пост принадлежит пользователю), мы будем использовать **полиморфные** отношения «один-ко-многим», где `Post` имеет полиморфно связанную модель (не обязательно модель `User`).

Сравним обычные и полиморфные отношения.

Определение обычного отношения «один-ко-многим»:

```php
// Модель `Post`.
class Post extends Model
{
  public function author()
  {
    return $this->belongsTo(User::class);
  }
}

// Модель `User`.
class User extends Model
{
  public function posts()
  {
    return $this->hasMany(Post::class);
  }
}
```

Определение полиморфного отношения «один-ко-многим»:

```php
// Модель `Post`.
class Post extends Model
{
  public function author()
  {
    return $this->morphTo();
  }
}

// Модель `User` (или аналогичная).
use JohnDoe\BlogPackage\Models\Post;

class Admin extends Model
{
  public function posts()
  {
    return $this->morphMany(Post::class, 'author');
  }
}
```

После добавления этого метода `author()` в нашу модель `Post` нам нужно обновить наш файл `create_posts_table_migration.php.stub`, чтобы отразить наши полиморфные отношения. Поскольку мы назвали метод `author`, то Laravel ожидает поля `author_id` и `author_type`. Последний содержит строку пространства имен модели, на которую мы ссылаемся, например, `App\User`.

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('body');
    $table->unsignedBigInteger('author_id');
    $table->string('author_type');
    $table->timestamps();
});
```

Теперь нам нужен способ предоставить нашему конечному пользователю возможность разрешить конкретным моделям иметь связь с нашей моделью `Post`. **Трейты** предлагают отличное решение именно для этой цели.

### Предоставление трейта

Создайте папку `Traits` в каталоге `src/` и добавьте следующий трейт `HasPosts`:

```php
// 'src/Traits/HasPosts.php'
<?php

namespace JohnDoe\BlogPackage\Traits;

use JohnDoe\BlogPackage\Models\Post;

trait HasPosts
{
  public function posts()
  {
    return $this->morphMany(Post::class, 'author');
  }
}
```

Теперь конечный пользователь может добавить оператор `use HasPosts` к любой из своих моделей, вероятно, модель `User`, который автоматически зарегистрирует отношение «один-ко-многим» с нашей моделью `Post`. Это позволит создавать новые посты следующим образом:

```php
// Учитывая, что у нас есть модель `User`, использующая трейт `HasPosts`
$user = User::first();

// мы можем создать новый пост из отношений.
$user->posts()->create([
  'title' => 'Some title',
  'body' => 'Some body',
]);
```

### Тестирование полиморфных отношений

Конечно, мы хотим доказать, что любая модель, использующая трейт `HasPost`, может создавать новые посты и, что эти посты будут корректно сохранены.

Поэтому мы создадим новую модель `User`, но не в каталоге `src/Models/`, а в нашем каталоге `tests/`.

Чтобы создавать пользователей в наших тестах, нам нужно перезаписать `UserFactory` пакета Orchestra Testbench, как показано ниже:

```php
// 'tests/UserFactory.php'
<?php

namespace JohnDoe\BlogPackage\Tests;

use Orchestra\Testbench\Factories\UserFactory as TestbenchUserFactory;

class UserFactory extends TestbenchUserFactory
{
  protected $model = User::class;

    /**
     * Определить состояние модели по умолчанию.
     *
     * @return array
     */
    public function definition()
    {
        return [
            'name' => $this->faker->name,
            'email' => $this->faker->unique()->safeEmail,
            'email_verified_at' => now(),
            'password' => bcrypt('password'),
            'remember_token' => \Illuminate\Support\Str::random(10),
        ];
    }
}
```

В модели `User` мы будем использовать те же трейты, что и в модели `User`, которая содержится в стандартном проектом Laravel, чтобы максимально приблизиться к реальному сценарию. Также мы используем собственный трейт `HasPosts` и `UserFactory`:

```php
// 'tests/User.php'
<?php

namespace JohnDoe\BlogPackage\Tests;

use Illuminate\Auth\Authenticatable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Foundation\Auth\Access\Authorizable;
use Illuminate\Contracts\Auth\Authenticatable as AuthenticatableContract;
use Illuminate\Contracts\Auth\Access\Authorizable as AuthorizableContract;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use JohnDoe\BlogPackage\Traits\HasPosts;

class User extends Model implements AuthorizableContract, AuthenticatableContract
{
    use HasPosts, Authorizable, Authenticatable, HasFactory;

    protected $guarded = [];

    protected $table = 'users';

    protected static function newFactory()
    {
        return UserFactory::new();
    }
}
```

Теперь, когда у нас есть модель `User`, нам также нужно добавить новую миграцию (стандартную миграцию таблицы пользователей, которая поставляется с Laravel) в наш каталог `database/migrations` как `create_users_table.php.stub`:

```php
// 'database/migrations/create_users_table.php.stub'
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateUsersTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->rememberToken();
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('users');
    }
}
```

Также загрузим миграцию в начале наших тестов, подключив файл миграции и выполнив метод `up()` в нашем `TestCase`:

```php
// 'tests/TestCase.php'
public function getEnvironmentSetUp($app)
{
    include_once __DIR__ . '/../database/migrations/create_posts_table.php.stub';
    include_once __DIR__ . '/../database/migrations/create_users_table.php.stub';

    // Выполнить метод `up()` импортированных классов миграции.
    (new \CreatePostsTable)->up();
    (new \CreateUsersTable)->up();
}
```

### Обновление фабрики нашей модели Post

Теперь, когда мы можем создавать модели `User` с помощью нашей новой фабрики, давайте создадим `User` в нашей `PostFactory`, а затем присвоим ему `author_id` и `author_type`:

```php
// 'database/factories/PostFactory.php'
<?php

namespace JohnDoe\BlogPackage\Database\Factories;

use JohnDoe\BlogPackage\Models\Post;
use Illuminate\Database\Eloquent\Factories\Factory;
use JohnDoe\BlogPackage\Tests\User;

class PostFactory extends Factory
{
    /**
     * Название модели соответствующей фабрики.
     *
     * @var string
     */
    protected $model = Post::class;

    /**
     * Определить состояние модели по умолчанию.
     *
     * @return array
     */
    public function definition()
    {
        $author = User::factory()->create();

        return [
            'title'     => $this->faker->words(3, true),
            'body'      => $this->faker->paragraph,
            'author_id' => $author->id,
            'author_type' => get_class($author)
        ];
    }
}
```

Затем мы обновляем модульный тест `Post`, чтобы убедиться, что можно присвоить `author_type`:

```php
// 'tests/Unit/PostTest.php'
class PostTest extends TestCase
{
  // Другие тесты ...

  /** @test */
  function a_post_has_an_author_type()
  {
    $post = Post::factory()->create(['author_type' => 'Fake\User']);
    $this->assertEquals('Fake\User', $post->author_type);
  }
}
```

Наконец, нам нужно убедиться, что наш тестовый `User` может создать `Post` и что он корректно сохраняется.

Поскольку мы не создаем новый пост, используя вызов определенного маршрута в приложении, то давайте сохраним этот тест в модульном тесте `Post`. В следующем разделе [Маршруты, контроллеры и шаблоны](09-routing.md) мы сделаем POST-запрос к конечной точке, чтобы создать новую модель `Post` и, следовательно, перейти к функциональному тестированию.

Модульный тест, который проверяет желаемое поведение между `User` и `Post`, может выглядеть следующим образом:

```php
// 'tests/Unit/PostTest.php'
class PostTest extends TestCase
{
  // Другие тесты ...

  /** @test */
  function a_post_belongs_to_an_author()
  {
    // Учитывая, что у нас есть автор
    $author = User::factory()->create();
    // и у этого автора есть пост
    $author->posts()->create([
        'title' => 'My first fake post',
        'body'  => 'The body of this fake post',
    ]);

    $this->assertCount(1, Post::all());
    $this->assertCount(1, $author->posts);

    // Используя `tap()`, создадим псевдоним `$post` для `$author->posts()->first()`,
    // чтобы обеспечит читабельность сгруппированных утверждений.
    tap($author->posts()->first(), function ($post) use ($author) {
        $this->assertEquals('My first fake post', $post->title);
        $this->assertEquals('The body of this fake post', $post->body);
        $this->assertTrue($post->author->is($author));
    });
  }
}
```

На этом этапе все тесты должны быть пройдены.
