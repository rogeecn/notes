---
title: "简单的 11 步在 Laravel 中实现测试驱动开发 "
date: 2021-11-26T16:19:29+08:00
draft: false
tags: ["Laravel", "Testing"]
---

**测试驱动开发**（英语：Test-driven development，缩写为 TDD）是一种[软件开发过程](https://zh.wikipedia.org/wiki/%E8%BD%AF%E4%BB%B6%E5%BC%80%E5%8F%91%E8%BF%87%E7%A8%8B)中的应用方法，由[极限编程](https://zh.wikipedia.org/wiki/%E6%9E%81%E9%99%90%E7%BC%96%E7%A8%8B)中倡导，以其倡导先写测试程序，然后编码实现其功能得名。

下文是我在 Medium 上看到的一篇文章讲述了如何在 Laravel 中实现测试驱动开发，我自己已经在项目中实现了测试驱动开发，并且通过持续集成完成了项目的自动测试，文章中有些我认为需要改进的地方也注明了，希望这篇文章能帮助到那些想写测试用例但是一直觉得无从下手的开发者们。文章一共有四篇，之后计划都翻译出来给大家参考。

原文链接: [https://medium.com/@jsdecena/simple-tdd-in...](https://medium.com/@jsdecena/simple-tdd-in-laravel-with-11-steps-c475f8b1b214)
<!--more-->
### 前言

大多数 Web 开发工程师第一次听到 TTD (test driven development) 测试驱动开发的时候都会感到畏惧，我也一样。当你刚开始试着进行测试驱动开发时你会感到很崩溃，但是如果你抗拒它就很难真正学会并掌握它。在本文中我会介绍如何在 Laravel 中进行测试驱动开发。

> Note: 这篇文章时针对 API 响应的 TDD. 如果你想看与 Laravel blade 相关的功能测试请看这篇文章 (还未来得及翻译，访问需要科学上网), [head up on this article.](https://medium.com/@jsdecena/crud-feature-testing-in-laravel-5-865b7532439)

### step 1: 准备 Laravel 测试套件

在项目根目录中，更新 `phpunit.xml` 文件的如下项目:

```xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
<env name="API_DEBUG" value="false"/>
<ini name="memory_limit" value="512M" />
```

更新完后 `phpunit.xml` 文件会像下面这样:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<phpunit backupGlobals="false"
  backupStaticAttributes="false"
  bootstrap="vendor/autoload.php"
  colors="true"
  convertErrorsToExceptions="true"
  convertNoticesToExceptions="true"
  convertWarningsToExceptions="true"
  processIsolation="false"
  stopOnFailure="false">
  <testsuites>
    <testsuite name="Feature">
      <directory suffix="Test.php">./tests/Feature</directory>
    </testsuite>

    <testsuite name="Unit">
      <directory suffix="Test.php">./tests/Unit</directory>
    </testsuite>
  </testsuites>

  <filter>
    <whitelist processUncoveredFilesFromWhitelist="true">
      <directory suffix=".php">./app</directory>
    </whitelist>
  </filter>
  
  <php>
    <env name="APP_ENV" value="testing"/>
    <env name="CACHE_DRIVER" value="array"/>
    <env name="SESSION_DRIVER" value="array"/>
    <env name="QUEUE_DRIVER" value="sync"/>
    <env name="DB_CONNECTION" value="sqlite"/>
    <env name="DB_DATABASE" value=":memory:"/>
    <env name="APP_DEBUG" value="false"/>
    <env name="MAIL_DRIVER" value="log"/>
    <ini name="memory_limit" value="512M" />
  </php>

</phpunit>
```

我们只需要在内存中进行测试这样测试运行的速度会快一些，所以在 database 配置项目中我们将使用 `sqlite` 和`:memory:` (Sqlite 的内存数据库)。 将 `APP_DEBUG` 设置为 false 因为我们只需要对真实产生的错误进行断言。随着项目迭代测试用例会越来越多所以在将来你可能会需要增加 `memory_limit` 的值。

***译者注: APP_DEBUG 建议不要改成 false，这样有助于让我们的代码写的更严谨***

在 Laravel 里测试用例的基类 `TestCase` 中作一些测试相关的准备:

```php
<?php
namespace Tests;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Illuminate\Foundation\Testing\DatabaseTransactions;
use Illuminate\Foundation\Testing\TestCase as BaseTestCase;
use Faker\Factory as Faker;
/**
 * Class TestCase
 * @package Tests
 * @runTestsInSeparateProcesses
 * @preserveGlobalState disabled
 */
abstract class TestCase extends BaseTestCase
{
    use CreatesApplication, DatabaseMigrations, DatabaseTransactions;
    protected $faker;
    /**
     * Set up the test
     */
    public function setUp()
    {
        parent::setUp();
        $this->faker = Faker::create();
    }
    /**
     * Reset the migrations
     */
    public function tearDown()
    {
        $this->artisan('migrate:reset');
        parent::tearDown();
    }
}
```

我们需要在 `TestCase` 中 use `DatabaseMigrations` 这个 trait 这样在执行每个测试用例时，迁移文件都会被执行一遍，于此同时在 `setUp()` 和 `tearDown()` 方法中我们需要执行创建测试环境和清理测试环境的相关操作。

***译者注： DatabaseMigrations 这个性状在测试 setUp 阶段会执行 migrate:refresh, 清除所有表然后重新执行迁移， 其实我们重置数据库后需要通过 seeder 来填充测试数据，所以我更倾向于不使用这个迁移而是在 TestCase 的 setUp 中 使用 migrate:refresh —seeder 来完成重置数据库和填充测试数据的操作***

### Step2：写测试用例

就像鲍勃大叔说的那样：” 除非先写测试用例否则你没有权利去写实现代码（implementation）“。现在开始写我们的测试吧。（测试驱动开发的第一天原则)

> 为了让 `phpunit` 识别测试，需要在测试方法上添加 / **@test** / 注释，或者是测试方法命名以 `test` 前缀开头。

```php
<?php
namespace Tests\Unit;
use Tests\TestCase;
class ArticleApiUnitTest extends TestCase
{
  public function it_can_create_an_article()
  {
      $data = [
        'title' => $this->faker->sentence,
        'content' => $this->faker->paragraph
      ];

      $this->post(route('articles.store'), $data)
        ->assertStatus(201)
        ->assertJson($data);
  }
}
```

在这个测试中，测试了是否能创建一篇文章，我们断言了在创建文章成功后应用将返回 201 状态码还有预期的 JSON 数据。

在创建好我们的第一个测试后，执行 `phpunit` 或者 `vendor/bin/phpunit`



![](https://cdn.learnku.com/uploads/images/201912/21/6964/MWNcOIbLx3.png!large)

当我们执行 `phpunit` 后测试结果显示失败了，这很正常因为在测试驱动开发中我们是先写测试程序，然后在编码实现功能的，所以在创建测试程序伊始测试程序执行后的结果就是测试失败 (测试驱动开发的第二条原则)。在测试中我们断言应用会返回 201 状态码但是却返回了 404，为什么？因为测试中请求的 URL 还未在应用中创建。这个 `api/v1/articles` POST 路由在应用中并不存在所以针对这个请求应用抛出了 404 错误。

接下来我们该作什么？

### Step 3: 在路由文件中创建测试里请求的 URL

让我们创建测试里请求的这个 URL 看看接下来会发生什么。

在你项目中的 `routes/api.php` 文件中创建这个 URL，在 api 中的路由其 URL 会自动加上 `/api` 前缀。

```php
<?php
use App\Http\Controllers\Api\ArticlesApiController;
use Illuminate\Support\Facades\Route;
Route::group(['prefix' => 'v1'], function () {
  Route::resource('articles', ArticlesApiController::class);
});
```

你可以通过 `artisan` 命令创建这个资源控制器:

```Bash
php artisan make:controller Api/ArticlesApiController —-resource
```

也可以手动创建。POST 请求将会路由到 `ArticlesApiContorller` 的 `store` 方法

### Step 4: DEBUG 控制器

```php
<?php
namespace App\Http\Controllers\Api;
class ArticlesApiController extends Controller 
{
    public function store() {
        dd('success!');
    }
}
```

现在让我们运行测试程序，看看测试程序是否能够访问到应用的这一部分，再次执行 `phpunit` 后返回结果如下：

![](https://cdn.learnku.com/uploads/images/201912/21/6964/6wImak5GSq.png!large)

执行后的结果在终端中提示出了字符串 `success!` 这意味着测试程序成功地访问到了创建文章的 API。

### Step 5: 验证你的输入

不要忘记验证将要存储到数据库中的数据，所以现在我们创建一个 `CreateArticleRequest` 类来控制输入数据的验证。

```php
<?php
namespace App\Http\Controllers\Api;
class ArticlesApiController extends Controller 
{
    public function store(CreateArticleRequest $request) {
        dd('success!');
    }
}
```

这个请求类包含了数据验证的规则:

```php
<?php
namespace App\Articles\Requests;
use Illuminate\Foundation\Http\FormRequest;
class CreateArticleRequest extends FormRequest
{
    /**
     * Transform the error messages into JSON
     *
     * @param array $errors
     * @return \Illuminate\Http\JsonResponse
     */
    public function response(array $errors)
    {
        return response()->json($errors, 422);
    }
    /**
     * Determine if the user is authorized to make this request.
     *
     * @return bool
     */
    public function authorize()
    {
        return true;
    }
    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
            'title' => ['required'],
            'content' => ['required']
        ];
    }
}
```

这是一个很好的验证数据的方式，因为之后我们可以新建另外一个测试程序来测试是否能捕获到这些验证错误，不过为了文章的精简我会在另外一篇单独的文章里来讲述如何创建捕获错误的测试程序。

### Step 6: 返回刚才创建的数据

记住在相应中应该返回指定的 JSON 结构这样我们就知道新建的数据成功存储到了数据库中。所以我们应该返回新创建的文章对象来满足我们上面创建的测试程序。

```php
<?php
namespace App\Http\Controllers\Api;
class ArticlesApiController extends Controller 
{
    /**
    * @param CreateArticleRequest $request
    */
    public function store(CreateArticleRequest $request) {
      return Article::create($request->all());
    }
}
```

你可能已经已经注意到了我们还没有创建 `Article` 这个类，接下来让我们来创建这个 Model。

### Step 7: 创建模型类

```php
<?php
namespace App\Articles;
use Illuminate\Database\Eloquent\Model;
class Article extends Model 
{
  protected $fillable = [
    'title',
    'content'
  ];

}
```

在 Article 这个模型类中，你需要定义可填充和序列化时需要被隐藏的字段，一旦 Article 类定义好后，返回到控制器中导入这个 Article 类。

```PHP
<?php
namespace App\Http\Controllers\Api;
use App\Articles\Article;
class ArticlesApiController extends Controller 
{
    /**
    * @param CreateArticleRequest $request
    */
    public function store(CreateArticleRequest $request) {
      return Article::create($request->all());
    }
}
```

我们差不多快要完成了！测试程序正确建立好了，URL 已经创建并且能够访问了，处理 URL 请求的控制器程序还有与数据表对应的模型类也都已经就绪了，现在让我们再试着执行一次 `phpunit` 命令。

### Step 8: 再次执行 phpunit 看看结果会怎么样[#](#135f2d)

  ![](https://user-gold-cdn.xitu.io/2018/7/16/164a33a1e816c0ef?w=800&h=300&f=png&s=151968)

它再一次失败了，这是好事还是坏事？可以说即是好事也是坏事。好的一方面是测试中断言会返回 201 状态码的 URL 的返回结果从之前的 404 错误变成了 500 错误 (如果你注意到了)。

不好的一方面是，它测试失败了我们需要让程序能够正确通过测试程序。当我们想要 debug 时我们想要看看应用程序到底抛出了什么错误。在 Laravel 测试程序中你只需要在发出 POST 请求后在调用 `dump()` 方法就能够看到应用程序返回的响应。

```php
<?php
namespace Tests\Unit;
use Tests\TestCase;
class ArticleApiUnitTest extends TestCase
{
  public function it_can_create_an_article()
  {
      $data = [
        'title' => $this->faker->sentence,
        'content' => $this->faker->paragraph
      ];

      $this->post(route('articles.store'), $data)
        ->dump()
        ->assertStatus(201)
        ->assertJson($data);
  }
}
```

你可以进一步 debug POST 请求后的输出，没准会得到更多你需要的信息，如果没有明确地给出提示发生了什么才导致的这个错误你可以去 Laravel 应用的日志文件 **/storage/logs/laravel.log** 里去查找错误信息。

现在让我们检查一下为什么会返回 500 错误。

Well， 因为我们在请求中正在尝试向一个不存在的数据表中写入数据所以才请求才会返回 500 错误。

### Step 9: 创建数据表

执行下面的 laravel `artisan` 命令：

```PHP
php artisan make:migration create_articles_table –create=articles
```

Laravel 会自动在 `/database/migrations` 里来创建迁移文件。

```php
<?php
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;
class CreateArticlesTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('articles', function (Blueprint $table) {
            $table->increments('id');
            $table->string('title');
            $table->text('content');
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
        Schema::dropIfExists('articles');
    }
}
```

默认迁移创建的数据表中只有 `id` 和时间字段，你需要根据需求添加需要的字段。

### Step 10: 再一次运行 phpunit

我们快要大功告成了，文章开始时我们承诺了完成 TDD 有 11 个步骤，现在是步骤 10 了！你需要轻轻拍下你的被来鼓励自己已经走到了这里。

![](https://user-gold-cdn.xitu.io/2018/7/16/164a33a1e8242eb9?w=800&h=304&f=png&s=151019)

Ooops, 又失败了？Whyyyyyyy? 如果你仔细检查的话你会发现你处在正确的轨道上！响应的状态码从 500 又变成 200。这意味着在发出 POST 请求后应用返回了执行成功的响应！这是没有匹配我们的需求，我们需要应用返回的响应状态码为 201 来让我们知道文章数据已经被正确写入到了数据库中。所以我们只需要修改一下我们的控制器程序为:

```php
<?php
namespace App\Http\Controllers\Api;
use App\Articles\Article;
class ArticlesApiController extends Controller 
{
    /**
    * @param CreateArticleRequest $request
    */
    public function store(CreateArticleRequest $request) {
      return response(Article::create($request->all()), 201);
    }
}
```

### Step 11: 运行 phpunit 然后祈祷一切会好[#](#b82507)

![](https://cdn.learnku.com/uploads/images/201912/21/6964/8KP55xVu5V.png!large)

恭喜大功告成！你成功让程序通过了测试，这就是测试驱动开发的第三条原则。

这就是测试驱动开发 (TDD) 在 Laravel 中的简单实现。还有其它的一些方法在这篇文章中没有来得及讲到，我在未来可能会那些，像是 `repository pattern` 等等。`repository pattern` 是 DDD (domain driven development) 领域驱动开发的最佳实践。

测试驱动开发还有很多方法这里没有设计到不过我觉得这篇文章已经足够让你开始试着在实践中应用测试驱动开发了。

***译者注***

通过文章总结起来测试驱动开发有三条原则：

1. 倡导先写测试程序，再编码实现功能。

2. 测试程序创建伊始肯定会测试失败。

3. 在让测试程序测试成功的过程中逐步编码实现功能