---
title: "Laravel - 从百草园到三味书屋"
date: 2021-12-08T09:55:03+08:00
draft: false
tags: ["Laravel", "阅读"]
---

网上偶得此书, 是 Laravel 的作者写的. Laravel 是近年来优秀的 PHP 框架, 国内外都有很多支持者. 该框架应用了大量 PHP5 尤其是 5.3 以后的新特性, 使得后端的开发进一步的简便而灵活. 具体可以看这里 http://www.laravel.com 作者写的这本书详细介绍了 Laravel 框架涉及的各种软件理念和工具, 如依赖注入, 控制反转容器, 面向接口编程等. 我读来收获颇丰, 所以希望翻译成中文以飨读者. 

## 作者自序

自从编写了 Laravel 框架之后, 我收到了大量让我写书的请求. 想让我出一本关于如何建造具有良好架构的复杂应用的指南. 由于每一个应用程序都是独特的, 这就需要此书讲述的是通用且实用的理论, 同时易于在各种项目中实施. 

因此我们将从基础要素之一的依赖注入开始讲起, 接着是深入了解服务提供者和应用程序结构, 以及"坚实"设计原则. 这些主题的中心思想会给你的 Laravel 项目奠定坚实的理论基础. 

如果你对 Laravel 上的高级架构有更进一步的问题的话, 或者想在本书看到更多没讲到的东西, 请给我发电子邮件. 我打算基于社区的反馈来进一步扩展本书, 所以你的意见很重要! 

最后, 十分感谢 Laravel 社区的每一个人. 你们为世界上成千上万的 PHP 开发者做出了巨大的贡献, 使得 PHP 开发变得更好玩更令人激动. 祝编码快乐! 
<!--more-->

## Dependency Injection 依赖注入

### The Problem 遇到的问题

Laravel 框架的基 础是一个功能强大的控制反转容器 (IoC container) . 为了真正理解本框架, 需要好好掌握该容器. 但我们要搞清楚, 控制反转容器只是一种用于方便实现"依赖注入"的工具. 要实现依赖注入并不一定需要控制反 转容器, 只是用容器会更方便和容易一点儿. 

首先来看看我们为何要使用依赖注入, 它能带来什么好处. 考虑下列代码: 

```php
class UserController extends BaseController{
    public function getIndex()
    {
        $users= User::all();
        return View::make('users.index', compact('users'));
    }
}
```

这段代码很 简短, 但我们要想测试这段代码的话就一定会和实际的数据库发生联系. 也就是说, Eloquent ORM (译者注: Laravel 的数据库对象模型库) 和该控制器有着紧耦合. 如果不使用 Eloquent ORM, 不连接到实际数据库, 我们就没办法运行或者测试这段代码. 这段代码同时也违背了"关注分离"这个软件设计原则. 简单讲: 这个控制器知道的太多了. 控制器不需要去了解数据是从哪儿来的, 只要知道如何访问就行. 控制器也不需要知道这数据是从 MySQL 或哪儿来的, 只需要知道这数据目前是可用的. 

> **关注分离**
> 每一个类都应该有单独的职责, 并且该职责应完全被这个类封装.  (译者注: 我认为就是不要让多个类负责同样的职责) 

关注分离的好处就是能让 Web 控制器和数据访问解耦. 这会使得实现存储迁移更容易, 测试也会更容易. "Web"就仅仅是为你真正的应用做数据的传输了. 

想象一下你有一个类似于监视器的程序, 有着很多线缆接口 (HDMI, VGA, DVI 等等) . 你可以通过不同的接口访问不同的监视器. 把 Internet 想象成另一个插进你程序线缆接口. 大部分显示器的功能是与线缆接口互相独立的. 线缆接口只是一 种传输机制就像 HTTP 是你程序的一种传输机制一样. 所以我们不想把传输机制 (控制器) 和业务逻辑混在一起. 这样的好处是很多其他的传输机制比如 API 调 用, 移动应用等都可以访问我们的业务逻辑. 

那么我们就别再将控制器和 Eloquent ORM 耦合在一起了. 咱们注入一个资料库类. 

### 建立约定 (Contract)

首先我们定义一个接口, 然后实现该接口. 

```php
interface UserRepositoryInterface
{
    public function all();
}

class DbUserRepository implements UserRepositoryInterface
{
    public function all()
    {
        return User::all()->toArray();
    }
}
```

然后我们将该接口的实现注入我们的控制器. 

```php
class UserController extends BaseController
{
    public function __construct(UserRepositoryInterface $users)
    {
        $this->users = $users;
    }

    public function getIndex()
    {
        $users=$this->users->all();
        return View::make('users.index', compact('users'));
    }
}
```

现在我们的控制器就完全和数据层面无关了. 在这里无知是福! 我们的数据可能来自 MySQL, MongoDB 或者 Redis. 我们的控制器不知道也不需要知道他们的区别. 仅仅做出了这么小小的改变, 我们就可以独立于数据层来测试 Web 层了, 将来切换存储实现也会很容易. 

>** 严守边界**
> 记得要保持清晰的责任边界. 控制器和路由是作为 HTTP 和你的应用程序之间的中间件来用的. 当编写大型应用程序时, 不要将你的领域逻辑混杂在其中 (控制器, 路由) . 

为了巩固学到的知识, 咱们来写一个测试案例. 首先, 我们要模拟一个资料库然后绑定到应用的 IoC 容器里. 然后, 我们要保证控制器正确的调用了这个资料库: 

```php
public function testIndexActionBindsUsersFromRepository()
{    
    // Arrange...
    $repository = Mockery::mock('UserRepositoryInterface');
    $repository->shouldReceive('all')->once()->andReturn(array('foo'));
    App::instance('UserRepositoryInterface', $repository);
    // Act...
    $response  = $this->action('GET', 'UserController@getIndex');

    // Assert...
    $this->assertResponseOk();
    $this->assertViewHas('users', array('foo'));
}
```

> Are you Mocking Me 你在模仿我么?
> 在上面的例子里, 我们使用了名为 `Mockery` 的模仿库. 这个库提供了一套整洁且富有表达力的方法, 用来模仿你写的类. Mockery 可以通过 Composer 安装. 

## Taking It Further 更进一步

让我们考虑另一个例子来巩固理解. 可能我们想要去提醒用户该交钱了. 我们会定义两个接口, 或者约定. 这些约定使我们在更改实际实现时更加灵活. 

```php
interface BillerInterface {
    public function bill(array $user, $amount);
}

interface BillingNotifierInterface {
    public function notify(array $user, $amount);
}
```

接下来我们要写一个 `BillerInterface` 的实现: 

```php
class StripeBiller implements BillerInterface{
    public function __construct(BillingNotifierInterface $notifier)
    {
        $this->notifier = $notifier;
    }
    public function bill(array $user, $amount)
    {
        // Bill the user via Stripe...
        $this->notifier->notify($user, $amount);
    }
}
```

只要遵守了每个类的责任划分, 我们很容易将不同的提示器 (notifier) 注入到账单类里面. 比如, 我们可以注入一个 `SmsNotifier` 或者 `EmailNotifier` . 账单类只要遵守了约定, 就不用再考虑如何实现提示功能. 只要是遵守约定 (接口) 的类, 账单类都能用. 这不仅仅是方便了我们的开发, 而且我们还可以通过模拟 `BillingNotifierInterface` 来进行无痛测试. 

### 使用接口
> 写接口可能看上去挺麻烦, 但实际上能加速你的开发. 你不用实现任何接口, 就能使用模拟库来模拟你的接口, 进而测试整个后台逻辑! 

那我们如何_做_依赖注入呢? 很简单: 

```php
$biller = new StripeBiller(new SmsNotifier);
```

这就是依赖注入. biller 不需再考虑提醒用户的事儿, 我们直接传给他一个提示器 (notifier) . 这种微小的改动能使你的应用焕然一新. 你的代码马上就变得更容易维护, 因为明确指定了类的职责边界. 并且更容易测试, 你只需使用模拟依赖即可. 

那 IoC 容器呢? 难道依赖注入不需要 IoC 容器么? 当然不需要! 在接下来的章节里面你会了解到, 容器使得依赖注入更易于管理, 但是容器不是依赖注入所必须的. 只要遵循本章提出的原则, 你可以在你任何的项目里面实施依赖注入, 而不必管该项目是否使用了容器. 

##  太像 Java 了? 

有人会说使用接口让 PHP 代码看上去太像 Java 了——即代码太罗嗦了——你必须定义接口然后实现它, 要多按好多下键盘. 

对于小而简单的应用来说, 以上说法也对. 接口通常是不必要的. 将代码耦合到那些你认为不会改变的地方也是可以的. 在你确信不会改变的地方就没有必要使用接口了. 架构师说"不会改变的地方是不存在的". 不过话说回来, 有时候的确不会改. 

在大型应用中接口是很有帮助的. 和提升的代码灵活性, 可测试性比起来, 多敲键盘费的功夫就微不足道了. 当你迅速的切换了代码实现的时候, 你的经理一定会被你的神速吓一跳的. 你也可以写出更适应变化的代码. 

总而言之, 记住本书提倡"简单"架构. 如果你在写小程序的时候无法遵守接口原则, 别觉得不好意思. 要记住做码农呢, 最重要就是开心. 如果你不喜欢写接口, 那就先简单的写代码吧. 日后再精进即可. 

# The IoC Container 控制反转容器  

## Basic Binding 基础绑定

我们已经学习了依赖注入, 接下来咱们一起来探索"控制反转容器"(IoC) . IoC 容器可以使你更容易管理依赖注入, Laravel 框架拥有一个很强大的 IoC 容器. Laravel 的核心就是这个 IoC 容器, 这个 IoC 容器使得框架各个组件能很好的在一起工作. 事实上 Laravel 的 Application 类就是继承自 Container 类! 

###  控制反转容器

> 控制反转容器使得依赖注入更方便. 当一个类或接口在容器里定义以后, 如何处理它们——如何在应用中管理, 注入这些对象? 

在 Laravel 应用里, 你可以通过 App 来访问控制反转容器. 容器有很多方法, 不过我们从最基础的开始. 让我们继续使用上一章写的 `BillerInterface` 和 `BillingNotifierInterface` , 且假设我们使用了 [Stripe](https://github.com/fabpot/pimple) 来进行支付操作. 我们可以将 Stripe 的支付实现绑定到容器里, 就像这样: 

```php
App::bind('BillerInterface', function()
{
    return new StripeBiller(App::make('BillingNotifierInterface'));
});
```

注意在我们处理 `BillingInterface` 时, 我们额外需要一个 `BillingNotifierInterface` 的实现, 也就是再来一个 bind: 

```php
App::bind('BillingNotifierInterface', function()
{
    return new EmailBillingNotifier;
});
```

如你所见, 这个容器就是个用来存储各种绑定的地方 (译者注: 这么理解简单点. 再扯匿名函数, 闭包就扯远了) . 一旦一个类在容器里绑定了以后, 我们可以很容易的在应用的任何位置调用它. 我们甚至可以在 bind 函数内写另外的 bind. 

### Have Acne?
> Laravel 框架的 Illuminate 容器和另一个名为 [Pimple](https://github.com/fabpot/pimple) 的 IoC 容器是可替换的. 所以如果你之前用的是 Pimple, 你尽可以大胆的升级为 [Illuminate Container](https://github.com/jilluminate/container), 后者还有更多新功能! 

一旦我们使用了容器, 切换接口的实现就是一行代码的事儿. 比如考虑以下代码: 

```php
class UserController extends BaseController{

public function __construct(BillerInterface $biller)
{
    $this->biller = $biller;
}
}
```

当这个控制器通被容器实例化后, 包含着 `EmailBillingNotifier` 的 `StripeBiller` 会被注入到这个控制器中 (译者注: 见上文的两个 bind) . 如果我们现在想要换一种提示方式, 我们可以简单的将代码改为这样: 

```
App::bind('BillingNotifierInterface', function()
{
    return new SmsBillingNotifier;
});
```

现在不管在应用的哪里需要一个提示器, 我们总会得到 `SmsBillingNotifier` 的对象. 利用这种结构, 我们的应用可以在不同的实现方式之间快速切换. 

只改一行就能切换代码实现, 这可是很厉害的能力. 比如我们想把短信服务从原来的提供商替换为 Twilio. 我们可以开发一个新的 Twilio 的提示器类 (译者注: 当然要继承自 `BillingNotifierInterface` ) 然后修改绑定语句. 如果 Twilio 有任何闪失, 我们只需修改一行代码就可以快速的切换回原来的短信提供商. 看到了吧, 依赖注入的好处多得很呢. 你能再想出几个使用依赖注入和控制反转容器的好处么? 

想在应用中只实例化某类一次? 没问题, 使用 `singleton` 方法吧: 

```php
App::singleton('BillingNotifierInterface', function()
{
    return new SmsBillingNotifier;
});
```

这样只要这个容器生成了这个提示器对象一次, 在接下来的生成请求中容器都只会提供这同样的一个对象. 

容器的 `instance` 方法和 `singleton` 方法很类似, 区别是 `instance` 可以绑定一个已经存在的对象. 然后容器每次返回的都是这个对象了. 

```php
$notifier = new SmsBillingNotifier;
App::instance('BillingNotifierInterface', $notifier);
```

现在我们熟悉了容器的基础用法, 让我们深入发掘它更强大的功能: 依靠反射来处理类和接口. 

### Stand Alone Container 容器独立运行
> 你的项目没有使用 Laravel? 但你依然可以使用 Laravel 的 IoC 容器! 只要用 Composer 安装了 `illuminate/container` 包就可以了. 

## Reflect Resolution 反射解决方案

用反射来自动处理依赖是 Laravel 容器的一个最强大的特性. 反射是一种运行时探测类和方法的能力. 比如, PHP 的 `ReflectionClass` 可以探测一个类的方法. `method_exists` 某种意义上说也是一种反射. 我们来把玩一下 PHP 的反射类, 试试下面的代码吧 (StripeBiller 换成你自己定义好的类) : 

```php
$reflection = new ReflectionClass('StripeBiller');
var_dump($reflection->getMethods());
var_dump($reflection->getConstants());
```

依靠这个强大的 PHP 特性, Laravel 的 IoC 容器可以实现很有趣的功能! 考虑接下来这个类: 

```php
class UserController extends BaseController
{
    public function __construct(StripBiller $biller)
    {
        $this->biller = $biller;
    }
}
```

注意这个控制器的构造函数暗示着有一个 `StripBiller` 类型的参数. 使用反射就可以检测到这种类型暗示. 当 Laravel 的容器无法解决一个类型的明显绑定时, 容器会试着使用反射来解决. 程序流程类似于这样的: 

1. 已经有一个 `StripBiller` 的绑定了么? 
2. 没绑定? 那用反射来探测一下 `StripBiller` 吧. 看看他都需要什么依赖. 
3. 解决 `StripBiller` 需要的所有依赖 (递归处理) 
4. 使用 `ReflectionClass->newInstanceArgs()` 来实例化 `StripBiller` 

如你所见, 容器替我们做了好多重活, 这能帮你省去写大量绑定的麻烦. 这就是 Laravel 容器最强大也是最独特的特性. 熟练掌握这种能力对构建大型 Laravel 应用是十分有益的. 

下面我们修改一下控制器, 改成这样会发生什么事儿呢? 

```php
class UserController extends BaseController
{
    public function __construct(BillerInterface $biller)
    {
        $this->biller = $biller;
    }
}
```
假设我们没有为 `BillerInterface` 做任何绑定, 容器该怎么知道要注入什么类呢? 要知道, interface 不能被实例化, 因为它只是个约定. 如果我们不提供更多信息的话, 容器是无法实例化这个依赖的. 我们需要明确指出哪个类要实现这个接口, 这就需要用到 `bind` 方法: 

```php
App::bind('BillerInterface', 'StripBiller');
```
这里我们只传了一个字符串进去, 而不是一个匿名函数. 这个字符串告诉容器总是使用 `StripBiller` 来作为 `BillerInterface` 的实现类. 此外我们也获得了只改一行代码即可轻松改变实现的能力. 比如, 假设我们需要切换到 Balanced Payments 作为我们的支付提供商, 我们只需要新写一个 `BalancedBiller` 来实现 `BillerInterface` 接口, 然后这样修改容器代码: 

```php
App::bind('BillerInterface', 'BalancedBiller');
```

我们的应用程序就装载上了的新支付实现代码了! 

你也可以使用 `singleton` 方法来实现单例模式. 

```php
App::singleton('BillerInterface', 'StripBiller');
```

### Master The Container 掌握容器

> 想了解更多关于容器的知识? 去读源码! 容器只有一个类 `Illuminate\Container\Container` . 读完了你就对容器有更深的认识了. 

# Interface As Contract 接口约定

## Strong Typing & Water Fowl 强类型和小鸭子

在之前的章节里, 涵盖了依赖注入的基础知识: 什么是依赖注入; 如何实现依赖注入; 依赖注入有什么好处. 之前章节里面的例子也模拟了将_interface_注入到_classes_里面的过程. 在我们继续学习之前, 有必要深入讲解一下接口, 而这正是很多 PHP 开发者所不熟悉的. 

在我成为 PHP 程序员之前, 我是写.NET 的. 你觉得我是 M 么? 在.NET 里可到处都是接口. 事实上很多接口是定义在.NET 框架核心中了, 一个好的理由是: 很多.NET 语言比如 C#和 VB.NET 都是_强类型的_. 也就是说, 你在给一个函数传值, 要么传原生类型对象, 要么就必须给这个对象一个明确的_类型_定义. 比如考虑以下 C#方法: 

```c#
public int BillUser(User user)
{
    this.biller.bill(user.GetId(), this.amount)
}
```

注意在这里, 我们不仅要定义传进去的参数是什么类型的, 还要定义这个方法返回值是什么类型的. C#鼓励类型安全. 除了指定的 `User` 对象, 它不允许我们传递其他类型的对象到 `BillUser` 方法中. 
然而 PHP 是一种_鸭子类型_的语言. 所谓鸭子类型的语言, 一个对象可用的方法取决于使用方式, 而非这个方法从哪儿继承或实现. 来看个例子: 

```php
public function billUser($user)
{
    $this->biller->bill($user->getId(), $this->amount);
}
```

在 PHP 里面, 我们不必告诉一个方法需要什么类型的参数. 实际上我们传递任何类型的对象都可以, 只要这个对象能响应 `getId` 的调用. 这里有个关于鸭子类型 (下文译作: 弱类型) 的解释: 如果一个东西看起来像个鸭子, 叫声也像鸭子叫, 那他就是个鸭子. 换言之在程序里, 一个对象看上去是个 User, 方法响应也像个 User, 那他就是个 User. 

不过 PHP 到底有没有_任何_强类型功能呢? 当然有! PHP 混合了强类型和弱类型的结构. 为了说明这点, 咱们来重写一下 `billUser` 方法: 

```php
public function billUser(User $user)
{
    $this->biller->bill($user->getId(), $amount);
}
```

给方法加上了加上了 `User` 类型提示后, 我们可以确信的说所有传入 `billUser` 方法的参数, 都是 `User` 类或是继承自 `User` 类的一个实例. 

强类型和弱类型各有优劣. 在强类型语言中, 编译器通常能提供编译时错误检查的功能, 这功能可是非常有用的. 方法的输入和输出也更加明确. 

与此同时, 强类型的特性也使得程序僵化. 比如 Eloquent ORM 中, 类似 `whereEmailOrName` 的动态方法就不可能在 C#这样的强类型语言里实现. 我们不讨论强类型弱类型哪种更好, 而是要记住他们分别的优劣之处. 在 PHP 里面使用强类型标记不是错误, 使用弱类型特性也不是错误. 但是不加思索, 不管实际情况去使用一种模式, 这么固执的使用就是错的. 

## A Contract Example 约定的范例

接口就是约定. 接口不包含任何代码实现, 只是定义了一个对象应该实现的一系列方法. 如果一个对象实现了一个接口, 那么我们就能确信这个接口所定义的一系列方法都能在这个对象上使用. 因为有约定保证了特定方法的实现标准, 通过_多态_也能使类型安全的语言变得更灵活. 

### Poly what? 多什么态? 

> 多态含义很广, 其本质上是说一个实体拥有多种形式. 在本书中, 我们讲多态是一个接口有着多种实现. 比如 `UserRepositoryInterface` 可以有 MySQL 和 Redis 两种实现, 每一种实现都是 `UserRepositoryInterface` 的一个实例. 

为了说明在强类型语言中接口的灵活性, 咱们来写一个酒店客房预订的代码. 考虑以下接口: 

```php
interface ProviderInterface{
    public function getLowestPrice($location);
    public function book($location);
}
```

当用户订房间时, 我们需要将此事记录在系统里. 所以在 `User` 类里面写点方法: 

```php
class User{
public function bookLocation(ProviderInterface $provider, $location)
{
    $amountCharged = $provider->book($location);
    $this->logBookedLocation($location, $amountCharged);
}
```

因为我们写出了 `ProviderInterface` 的类型提示, 该 `User` 类的就可以放心大胆的认为 `book` 方法是可以调用的. 这使得 `bookLocation` 方法有了重用性. 当用户想要换一家酒店提供商时也就更灵活. 最后咱们来写点代码来强化他的灵活性. 

```php
$location = 'Hilton, Dallas';

$cheapestProvider = $this->findCheapest($location, array(
    new PricelineProvider, 
    new OrbitzProvider, 
));

$user->bookLocation($cheapestProvider, $location);
```

太棒了! 不管哪家是最便宜的, 我们都能够将他传入 `User` 对象来预订房间了. 由于 `User` 对象只需要要有一个符合 `ProviderInterface` 约定的实例就可以预订房间, 所以未来有更多的酒店供应商我们的代码也可以很好的工作. 

### Forget The Details 忘掉细节

> 记住, 接口实际上不真正做任何事情. 它只是简单的定义了类们**必须**实现的一系列方法. 

## Interfaces & Team Development 接口与团队开发

当你的团队在开发大型应用时, 不同的部分有着不同的开发速度. 比如一个开发人员在制作数据层, 另一个开发人员在做前端和网站控制器层. 前端开发者想测试他的控制器, 不过后端开发较慢没法同步测试. 那如果两个开发者能以接口的方式达成协议, 后台开发的各种类都遵循这种协议, 就像这样: 

```php
interface OrderRepositoryInterface {
    public function getMostRecent(User $user);
}
```

一旦建立了约定, 就算约定还没实现, 前端开发者也可以测试他的控制器了! 这样应用中的不同组件就可以按不同的速度开发, 并且单元测试也可以做. 而且这种处理方法还可以使组件内部的改动不会影响到其他不相关组件. 要记着无知是福. 我们写的那些类们不用知道别的类_如何_实现的, 只要知道它们_能_实现什么. 这下咱们有了定义好的约定, 再来写控制器: 

```php
class OrderController {
    public function __construct(OrderRepositoryInterface $orders)
    {
        $this->orders = $orders;
    }
    public function getRecent()
    {
        $recent = $this->orders->getMostRecent(Auth::user());
        return View::make('orders.recent', compact('recent'));
    }
}
```

前端开发者甚至可以为这接口写个"假"实现, 然后这个应用的视图就可以用假数据填充了: 

```php
class DummyOrderRepository implements OrderRepositoryInterface {
    public function getMostRecent(User $user)
    {
        return array('Order 1', 'Order 2', 'Order 3');
    }
}
```

一旦假实现写好了, 就可以被绑定到 IoC 容器里, 然后整个程序都可以调用他了: 

```php
App::bind('OrderRepositoryInterface', 'DummyOrderRepository');
```

接下来一旦后台开发者写完了真正的实现代码, 比如叫 `RedisOrderRepository` . 那么 IoC 容器就可以轻易的切换到真正的实现上. 整个应用就会使用从 Redis 读出来的数据. 

### Interface As Schematic 接口就是大纲
> 接口在开发程序的"骨架"时非常有用. 在设计组件时, 使用接口进行设计和讨论都是对你的团队有益处的. 比如定义一个 `BillingNotifierInterface` 然后讨论他有什么方法. 在写任何实现代码前先用接口讨论好一套好的 API! 

# Service Providers 服务提供者

## As Bootstrapper 他是引导程序

一个 Laravel 服务提供者就是一个用来进行 IoC 绑定的类. 事实上, Laravel 有好几十个服务提供者, 用于管理框架核心组件的容器绑定. 几乎框架里每一个组件的 IoC 绑定都是靠服务提供者来做的. 你可以在 `app/config/app.php` 这个文件里查看目前有哪些服务提供者. 

一个服务提供者必须有一个 `register` 方法. 你可以在这个方法里写 IoC 绑定. 当一个请求发过来, 程序框架刚启动时, 所有在你配置文件里的服务提供者的 `register` 方法就会被调用. 这在程序周期的很早的地方就会执行, 所以在你自己的引导代码 (比如那些在 `start` 目录里的文件) 里所有的服务已经准备好了. 

### Register Vs. Boot 注册 Vs 引导代码

> 永远不要在 `register` 方法里面使用任何服务. 该方法只是用来进行 IoC 绑定的地方. 所有关于绑定类后续的判断, 交互都要在 `boot` 方法里进行. 

你用 Composer 安装的一些第三方包也会有服务提供者. 在第三方包的安装说明里一般都会告诉你要在 `providers` 数组里加上一行. 一旦你加上了, 那这个服务就算安装好了. 

### Package Providers 包提供者

> 不是所有的第三方包都需要服务提供者. 事实上一个包并不需要服务提供者. 因为服务提供者只是一个用来自动初始化服务组件的地方, 一个方便管理引导代码和容器绑定的地方. 

### Deferred Providers 延迟加载的服务提供者

并非在你配置文件中的 `providers` 数组里的所有提供者在每次请求都会被实例化. 否则会对性能不利, 尤其是这个服务的功能用不到的情况下. 比如, `QueueServiceProvider` 服务就不是每次都用得到. 

为了达到只实例化需要的服务的提供者, Laravel 生成了"服务清单"并且储存在了 `app/storage/meta` 目录下. 这份清单列出了应用里所有的服务提供者, 包括容器绑定的名字也记录了. 这样, 当应用想让容器取出一个名为 `queue` 的绑定时, Laravel 知道需要先实例化并运行 `QueueServiceProvider` 因为在服务清单里记录着该服务提供者能提供 `queue` 的绑定. 如此这般框架就能够延迟加载每个请求需要的服务了, 性能大大提高. 

### Manifest Generation 如何生成服务清单

> 当你在 `providers` 数组里新增一条, Laravel 在下一次请求时就会自动重新生成服务清单. 

如果你有时间, 去看看服务清单文件里面的内容. 理解这个文件的结构有助于你对服务进行排错. 

## As Organizer 作为管理工具

想制作一个结构优美的 Laravel 应用的话, 就要去学习如何用服务提供者来管理代码. 当你在注册 IoC 绑定的时候, 所有代码都杂乱的塞进了 `app/start` 路径下的文件里. 别再这样做了, 使用服务提供者来注册这些吧. 

### Get It Started 万物之初

> 你应用的"启动"文件都储存在 `app/start` 目录下. 根据不同的请求入口, 系统会载入不同的启动文件. 在全局的 `start.php` 文件加载后, 系统会根据执行环境的不同来加载不同的启动文件. 此外, 在执行命令行程序时, `artisan.php` 文件会被载入. 

咱们来考虑这个例子. 也许我们的应用正在使用 [Pusher](http://pusher.com) 来为客户推送消息. 为了将我们的应用和 Pusher 解耦, 我们要定义 `EventPusherInterface` 接口和对应的实现类 `PusherEventPusher` . 这样在需求变化或应用改进时, 我们就可以随时轻松的改变推送服务提供商. 

```php
interface EventPusherInterface{
    public function push($message, array $data = array());
}

class PusherEventPusher implements EventPusherInterface{
    public function __construct(PusherSdk $pusher)
    {
        $this->pusher = $pusher;
    }
    public function push($message, array $data = array())
    {
        // Push message via the Pusher SDK...
    }
}
```

接下来我们创建一个 `EventPusherServiceProvider` : 

```php
use Illuminate\Support\ServiceProvider;

class EventPusherServiceProvider extends ServiceProvider {
    public function register()
    {
        $this->app->singleton('PusherSdk', function()
        {
            return new PusherSdk('app-key', 'secret-key');
        }

        $this->app->singleton('EventPusherInterface', 'PusherEventPusher');
    }
}

```

很好! 我们对事件推送进行了清晰的抽象, 同时我们也有了一个很不错的地方进行注册, 绑定其他相关的东西到容器里. 最后一步只需要将 `EventPusherServiceProvider` 写入 `app/config/app.php` 文件内的 `providers` 数组里就可以了. 现在这个应用里的 `EventPusherInterface` 已经被绑定到了正确的实现类上. 

### Should You Singleton? 要使用单例么? 

> 用不用单例可以这样来考虑: 如果在一次请求周期中该类只需要有一个实例, 就使用 `singleton` ; 否则就使用 `bind` . 

```php
App::singleton('EventPusherInterface', 'PusherEventPusher');
```

当然服务提供者的功能不仅仅局限于消息推送. 像是云存储, 数据库访问, 自定义的视图引擎比如 Twig 等等都可以用这种模式来设置. 服务提供者就是你的应用里的启动代码和管理工具, 没什么神奇的. 

所以大胆的去创建你自己的服务提供者. 并不是你非要发布个什么软件包才需要服务提供者, 他们只是非常好的管理代码的工具. 使用它们的力量去管理好应用中的各个组件吧. 

## Booting Providers 服务提供者的启动过程

在所有服务提供者都注册以后, 他们就进入了"启动"过程. 该过程会触发每个服务提供者的 `boot` 方法. 这里会发生一种常见的错误用法: 在 `register` 方法里面调用其他的服务. 由于在 `register` 方法里我们不能保证所有其他服务都已经被加载, 所以在该方法里调用别的服务有可能会出错. 所以如果你想在服务提供者里调用别的服务, 请在 `boot` 方法里做这种事儿. `register` 方法**只能**进行容器注册. 

在启动方法里面, 你想做什么都可以: 注册事件监听, 引入路由文件, 注册过滤器, 或者其他你能想象到的事儿. 再强调一下, 要发挥服务提供者的管理功能. 可能你想将相关的多个事件监听归为一组? 将他们放到一个服务提供者的 `boot` 方法里, 这会很管用的! 或者你也可以引入单独的"events", "routes"PHP 文件: 

```php
public function boot()
{
    require_once __DIR__.'/events.php';
    require_once __DIR__.'/routes.php';
}

```

我们已经学习了依赖注入以及如何使用服务提供者来组织管理我们的项目. 这样我们的 Laravel 应用就有了一个很好的基础, 它结构优美并且易于维护和测试. 接下来, 我们将探索 Laravel 框架本身是如何使用服务提供者的, 并且深究其原理! 

### Don't Box Yourself In 不要让条条框框限制你自己

> 记住, 服务提供者不仅仅是专业的软件包才能使用. 请大胆的使用它来组织管理你的应用服务吧. 

## Providing The Core 核心也是服务提供者的模式

你可能已经注意到, 在 `app` 配置文件里面已经有了很多服务提供者. 每一个都负责启动框架核心的一部分. 比如 `MigrationServiceProvider` 负责启动数据库迁移的类, 包括 Artisan 里面的命令. `EventServiceProvide` 负责启动和注册事件调度机制. 不同的服务提供者有着不同的复杂度, 但他们都负责启动核心的一部分. 

### Meet Your Providers 和服务提供者们见见面

> 理解 Laravel 核心的最好方法是去读它的核心服务源码. 如果你对这些服务的源码, 容器注册等都很熟悉, 那么你对 Laravel 是如何工作的将会有十分深刻的理解. 

大部分的服务提供者是延迟加载的, 意味着并非所有请求都会调用到他们; 然而有一些很基础的服务是每一次请求都会被加载的, 比如 `FilesystemServiceProvide` 和 `ExceptionServiceProvider` . 有人会说核心服务提供者和应用程序容器_就是_Laravel. Laravel 其实是将这么多不同部分联系起来, 形成一个单一的, 内聚的整体的这么一个机制. 拿建筑来比喻, 那些服务提供者就是框架的预制模块. 

正如之前提到的那样, 如果你想更深的了解框架是如何运行的, 请读 Lravel 的核心服务的源码吧. 读过之后, 你会对框架如何将各部分组合在一起, 每一个服务是如何为你所用这些机制有更坚实的理解. 此外, 有了这些进一步的理解, 你也可以为 Laravel 添砖加瓦! 

# Application Structure 应用结构

## Introduction 介绍

这个类要写到哪儿? 这是一个在用框架写应用程序时十分常见的问题. 大量的开发人员都有这个疑问. 他们被灌输"Model"就是"Database", 在控制器里面处理 HTTP 请求, 在模型里操作数据库, 视图里包含了要显示的 HTML. 不过, 发送电子邮件的类要写到哪儿? 数据验证的类要写到哪儿? 调用外部 API 的类要写到哪儿? 在这一章节, 我们将学习如何写结构优美的 Laravel 应用, 打破长久以来掣肘开发人员的普遍思维惯性这个拦路虎, 最终做出好的设计. 

## MVC Is Killing You MVC 是慢性谋杀

为了做出好的程序设计, 最大的拦路虎就是一个简单的缩写词: M-V-C. 模型, 视图, 控制器主宰了 Web 框架的思想已经好多年了. 这种思想的流行某种程度上是托了 Ruby on Rails 愈加流行的福. 然而, 如果你问一个开发人员"模型"的定义是什么. 通常你会听到他嘟哝着什么"数据库"之类的东西. 这么说, 模型_就是_数据库了. 不管这意味着什么, 模型里包含了关于数据库的_一切_. 但是, 你很快就会知道, 你的应用程序需要的不仅仅是一个简单的数据库访问类. 他需要更多的逻辑如: 数据验证, 调用外部服务, 发送电子邮件, 等等更多. 

### What Is A Model? 模型是啥? 

> 单词"model"的含义太模糊了, 很难说明白准确的含义. 更具体来讲, 模型是用来将我们的应用划分成更小, 更清晰的类, 使得各代码部分有着明确的权责. 

所以怎么解决这个问题 (译者注: 上文中"更多的业务逻辑") 呢? 很多开发者开始将业务逻辑包装到控制器里面. 当控制器庞大到一定规模, 他们将会需要重用业务逻辑. 大部分开发人员没有将这些业务逻辑提取到别的类里面, 而是错误的臆想他们_需要_在控制器里面调用别的控制器. 这种模式通常被称为"HMVC". 不幸的是, 这种模式通常也预示着糟糕的程序设计, 并且控制器已经太复杂了. 

### HMVC (Usually) Indicates Poor Design
>
### HMVC (通常) 预示着糟糕的设计. 
>
> 你觉得需要在控制器里面调用其他的控制器? 这通常预示着糟糕的程序设计并且你的控制器里面业务逻辑太多了. 把业务逻辑抽出来放到一个新的类里面, 这样你就可以在其他任何控制器里面调用了. 

有一种更好的程序结构. 但首先我们要忘掉以往我们被灌输的关于"模型"的一切. 干脆点, 让我们直接删掉 model 目录, 重新开始吧! 

## Bye, Bye Models 再见, 模型

删掉你的 `models` 目录了么? 还没删就赶紧删了! 我们将要在 `app` 目录下创建个新的目录, 目录名就以我们这个应用的名字来命名, 这次我们就叫 `QuickBill` 吧. 在后续的讨论中, 我们在前面写的那些接口和类都会出现. 

### Remember The Context 注意使用场景
>
> 记住, 如果你在写一个很小的 Laravel 应用, 那在 `models` 目录下写几个 Eloquent 模型其实挺合适的. 但在本章节, 我们主要关注如何开发更有合适"层次"架构的大型复杂项目. 

这样我们现在有了个 `app/QuickBill` 目录, 它和应用目录下的其他目录如 `controllers` 还有 `views` 都是平级的. 在 `QuickBill` 目录下我们还可以创建几个其他的目录. 我们来在里面创建个 `Repositories` 和 `Billing` 目录. 目录都创建好以后, 别忘了在 `composer.json` 文件里加入 PSR-0 的自动载入机制: 

```json
"autoload": {
"psr-0":    {
    "QuickBill":    "app/"
}
}
```

> 译者注: psr-0 也可以改成 psr-4, "psr-4": { "QuickBill\\": "app/QuickBill" } psr-4 是比较新的建议标准, 和 psr-0 具体有什么区别请自行检索. 

现在我们把继承自 Eloquent 的模型类都放到 `QuickBill` 目录下面. 这样我们就能很方便的以 `QuickBill\User` , `QuickBill\Payment` 的方式来使用它们. `Repositories` 目录属于 `PaymentRepository` 和 `UserRepository` 这种类, 里面包含了所有对数据的访问功能比如 `getRecentPayments` 和 `getRichestUser` . `Billing` 目录应当包含调用第三方支付服务 (如 Stripe 和 Balanced) 的类. 整个目录结构应该类似这样: 

```
// app
// QuickBill
    // Repositories
        -> UserRepository.php
        -> PaymentRepository.php
    // Billing
        -> BillerInterface.php
        -> StripeBiller.php
    // Notifications
        -> BillingNotifierInterface.php
        -> SmsBillingNotifier.php
    User.php
    Payment.php
```

### What About Validation 数据验证怎么办? 
> 在哪儿进行数据验证常常困扰着开发人员. 可以考虑将数据验证方法写进你的"实体"类里面 (好比 `User.php` 和 `Payment.php` ) . 方法名可以设为 `validForCreation` 或 `hasValidDomain` . 或者你也可以专门创建个验证器类 `UserValidator` , 放到 `Validation` 命名空间下, 然后将这个验证器类注入到你的 repository 类里面. 两种方式你都可以试试, 看哪个你更喜欢! 

摆脱了 `models` 目录后, 你通常就能克服心理障碍, 实现好的设计. 使得你能创建一个更合适的目录结构来为你的应用服务. 当然, 你建立的每一个应用程序都会有一定的相似之处, 因为每个复杂的应用程序都需要一个数据访问 (repository) 层, 一些外部服务层等等. 

### Don't Fear Directories 别害怕目录
>
> 不要惧怕建立目录来管理应用. 要常常将你的应用切割成小组件, 每一个组件都要有十分专注的职责. 跳出"模型"的框框来思考. 比如我们之前就说过, 你可以创建个 `Repositories` 目录来存放你所有的数据访问类. 

## It's All About The Layers 核心思想就是分层

你可能注意到, 优化应用的设计结构的关键就是责任划分, 或者说是创建不同的责任层次. 控制器只负责接收和响应 HTTP 请求然后调用合适的业务逻辑层的类. 你的业务逻辑/领域逻辑层_才是_你真正的程序. 你的程序包含了读取数据, 验证数据, 执行支付, 发送电子邮件, 还有你程序里任何其他的功能. 事实上你的领域逻辑层不需要知道任何关于"网络"的事情! 网络仅仅是个访问你程序的传输机制, 关于网络和 HTTP 请求的一切不应该超出路由和控制器层. 做出好的设计的确很有挑战性, 但好的设计也会带来可持续发展的清晰的好代码. 

举个例子. 与其在你业务逻辑类里面直接获取网络请求, 不如你直接把网络请求从控制器传给你的业务逻辑类. 这个简单的改动将你的业务逻辑类和"网络"分离开了, 并且不必担心怎么去模拟网络请求, 你的业务逻辑类就可以简单的测试了: 

```php
class BillingController extends BaseController{
    public function __construct(BillerInterface $biller)
    {
        $this->biller = $biller;
    }
    public function postCharge()
    {
        $this->biller->chargeAccount(Auth::user(), Input::get('amount'));
        return View::make('charge.success');
    }
}

```

现在 `chargeAccount` 方法更容易测试了. 我们把 `Request` 和 `Input` 从 `BillingInterface` 里提出来, 然后在控制器里把方法需要的支付金额直接传过去. 

编写拥有高可维护性应用程序的关键之一, 就是责任分割. 要时常检查一个类是否管得太宽. 你要常常问自己"这个类需不需要关心 XXX 呢? "如果答案是否定的, 那么把这块逻辑抽出来放到另一个类里面, 然后用依赖注入的方式进行处理.  (译者注: 依赖注入的不同方式还记得么? 调用方法传参, 构造函数传参, 从 IoC 容器获取等等. ) 

### Single Reason To Change
>
> 如何判断一个类是否管得太宽, 有一个有用的方法就是检查你为什么要改这块儿代码. 举个例子: 当我们想调整通知逻辑的时候, 我们需要修改 `Biller` 的实现代码么? 当然不需要, `Biller` 的实现仅仅需要考虑支付, 它与通知逻辑应当仅通过约定来进行交互. 使用这种思路过一遍代码, 会让你很快找出应用中需要改进的地方. 

## Where To Put "Stuff" 东西都放哪儿? 

当用 Laravel 开发应用时, 你可能迷惑于应该把各种"东西"都放在哪儿. 比如, 辅助函数要放在哪里? 事件监听器要放在哪里? 视图组件要放在哪里? 答案可能出乎你的意料——"想放哪儿都行! "Laravel 并没有很多在文件系统上的约定. 不过这个答案的确不能让人满意, 所以下面我们就这个问题展开讨论, 一起探索这些"东西"究竟可以放在哪儿. 

### Helper Functions 辅助函数

Laravel 有一个文件 ( `support/helpers.php` ) 里面都是辅助函数. 你或许希望创建一个类似的文件来存储你自己的辅助函数. "start"文件是个不错的入口, 该文件会在应用的每一次请求时被访问. 在 `start/global.php` 里, 你可以引入你自己写的 `helpers.php` 文件, 就像这样: 

```php
// Within app/start/global.php

require_once __DIR__.'/../helpers.php';
//译者注: 该 helpers.php 文件位于 app 目录下, 需要你自己创建. 你想放到别的地方也可以. 

```

### Event Listeners 事件监听器

事件监听器当然不该放到 `routes.php` 文件里面, 若直接放到"start"目录下的文件里会比较乱, 所以我们要找另外的地方来存放. 服务提供者是个好地方. 我们之前了解到, 服务提供者可不仅仅是用来做依赖注入绑定, 还可以干其他事儿. 可以将事件监听器用服务提供者来管理起来, 让代码更整洁, 不至于影响到你应用的主要逻辑代码. 视图组件其实和事件差不多, 也可以类似的放到服务提供者里面. 

例如使用服务提供者进行事件注册可以这样: 

```php
<?php 
namespace QuickBill\Providers;
use Illuminate\Support\ServiceProvider;
class BillingEventsProvider extends ServiceProvider{
    public function boot()
    {
        Event::listen('billing.failed', function($bill)
        {
            // Handle failed billing event...
        });
    }
}

```

创建好服务提供者后, 就可以将它加入到 `app/config/app.php` 配置文件的 `providers` 数组里. 

#### Wear The Boot 注意启动流程
>
> 记住在上面的例子里面, 我们在 `boot` 方法里进行编写是有原因的. `register` 方法**只能**用来进行依赖注入绑定. 

### Error Handlers 错误处理

如果你的应用里面有很多自定义的错误处理方法, 那你的"启动"文件可能会很臃肿. 和刚才的事件监听器一样, 错误处理方法也最好放到服务提供者里面. 这种服务提供者可以命名为像 `QuickBillErrorProvider` 这种. 然后你在 `boot` 方法里想注册多少错误处理方法都可以了. 重申一下精神: 让呆板的代码离你应用的业务逻辑越远越好. 下方展示了这种服务提供者的一种可能的书写方法: 

```php
<?php 
namespace QuickBill\Providers;
use App, Illuminate\Support\ServiceProvider;

class QuickBillErrorProvider extends ServiceProvider {
    public function register()
    {    
        //
    }

    public function boot()
    {
        App::error(function(BillingFailedException $e)
        {
            // Handle failed billing exceptions ...
        });
    }
}

```

#### The Small Solution 简便做法
>
> 当然如果你只有一两条简单的错误处理方法, 那么都写在"启动"文件里面也是一种又快又好的简便做法. 

### The Rest 其他

通常只要遵循 PSR-0 (译者注: 或 PSR-4) 就可以保持类的整洁. 命令式的代码比如事件监听器, 错误处理器还有其他"注册"性质的操作都可以放在服务提供者里面. 对于什么代码要放在什么地方这个问题, 结合你目前为止学到的知识, 应当可以给出一个有理有据的答案了. 但永远不要害怕试验. Laravel 最美妙之处就是你可以做出最适合你自己的风格. 去探索和发现最适合你自己应用的结构吧, 别忘了和他人分享你的见解! 

例如你可能注意到我们上面的例子, 你可以创建个 `Providers` 的命名空间来存放你自己写的服务提供者, 目录就类似于这样: 

```
// app
// QuickBill
    // Billing
    // Extensions
        //Pagination
            -> Environment.php
    // Providers
        -> EventPusherServiceProvider.php
    // Repositories
    User.php
    Payment.php

```

看上面的例子我们有 `Providers` 和 `Extensions` 两个命名空间 (译者注: 分别对应两个同名目录) . 你自己写的服务提供者可以放到 `Providers` 命名空间下. 那个 `Extensions` 命名空间可以用来存放你对框架核心进行扩展的类. 

# Applied Architecture: Decoupling Handlers 实用做法: 解耦处理函数

## Introduction 介绍

我们已经讨论了用 Laravel4 制作优美的程序架构的各个方面, 让我们再深入一些细节. 在本章, 我们将讨论如何解耦各种处理函数: 队列处理函数, 事件处理函数, 甚至其他"事件型"的结构如路由过滤器. 

### Don't Clog Your Transport Layer 不要堵塞传输层
>
> 大部分的"处理函数"可以被当作_传输层_组件. 也就是说, 队列触发器, 被触发的事件, 或者外部发来的请求等都可能调用处理函数. 可以把处理函数理解为控制器, 避免在里面堆积太多具体业务逻辑实现. 

## Decoupling Handlers 解耦处理函数

接下来我们看一个例子. 考虑有一个队列处理函数用来给用户发送手机短信. 信息发送后, 处理函数还要记录消息日志来保存给用户发送的消息历史. 代码应该看起来是这样: 

```php
class SendSMS{
    public function fire($job, $data)
    {
        $twilio = new Twilio_SMS($apiKey);
        $twilio->sendTextMessage(array(
            'to'=> $data['user']['phone_number'], 
            'message'=> $data['message'], 
        ));
        $user = User::find($data['user']['id']);
        $user->messages()->create(array(
            'to'=> $data['user']['phone_number'], 
            'message'=> $data['message'], 
        ));
        $job->delete();
    }
}

```

简单审查下这个类, 你可能会发现一些问题. 首先, 它难以测试. 在 `fire` 方法里直接使用了 `Twilio_SMS` 类, 意味着我们没法注入一个模拟的服务 (译者注: 即一旦测试则必须发送一条真实的短信) . 第二, 我们直接使用了 Eloquent, 导致在测试时肯定会对数据库造成影响. 第三, 我们没法在队列外面发送短信, 想在队列外面发还要重写一遍代码. 也就是说我们的短信发送逻辑和 Laravel 的队列耦合太多了. 

将里面的逻辑抽出成为一个单独的"服务"类, 我们即可将短信发送逻辑和 Laravel 的队列解耦. 这样我们就可以在应用的任何位置发送短信了. 我们将其解耦的过程, 也令其变得更易于测试. 

那么我们来稍微改一改: 

```php
class User extends Eloquent {
    /**
     * Send the User an SMS message
     *
     * [@param](https://my.oschina.net/u/2303379) SmsCourierInterface $courier
     * [@param](https://my.oschina.net/u/2303379) string $message
     * [@return](https://my.oschina.net/u/556800) SmsMessage
     */
    public function sendSmsMessage(SmsCourierInterface $courier, $message)
    {
        $courier->sendMessage($this->phone_number, $message);
        return $this->sms()->create(array(
            'to'=> $this->phone_number, 
            'message'=> $message, 
        ));
    }
}

```

在本重构的例子中, 我们将短信发送逻辑抽出到 `User` 模型里. 同时我们将 `SmsCourierInterface` 的实现注入到该方法里, 这样我们可以更容易对该方法进行测试. 现在我们已经重构了短信发送逻辑, 让我们再重写队列处理函数: 

```php
class SendSMS {
    public function __construct(UserRepository $users, SmsCourierInterface $courier)
    {
        $this->users = $users;
        $this->courier = $courier;
    }
    public function fire($job, $data)
    {
        $user = $this->users->find($data['user']['id']);
        $user->sendSmsMessage($this->courier, $data['message']);
        $job->delete();
    }
}

```

你可以看到我们重构了代码, 使得队列处理函数更轻量化了. 它本质上变成了队列系统和你_真正_的业务逻辑之间的_转换层_. 这可是很了不起! 这意味着我们可以很轻松的脱离队列系统来发送短信息. 最后, 让我们为短信发送逻辑写一些测试代码: 

```php
class SmsTest extends PHPUnit_Framework_TestCase {
    public function testUserCanBeSentSmsMessages()
    {
        /**
         * Arrage ...
         */
        $user = Mockery::mock('User[sms]');
        $relation = Mockery::mock('StdClass');
        $courier = Mockery::mock('SmsCourierInterface');

        $user->shouldReceive('sms')->once()->andReturn($relation);

        $relation->shouldReceive('create')->once()->with(array(
            'to' => '555-555-5555', 
            'message' => 'Test', 
        ));

        $courier->shouldReceive('sendMessage')->once()->with(
            '555-555-5555', 'Test'
        );

        /**
         * Act ...
         */
        $user->sms_number = '555-555-5555'; //译者注: 应当为 phone_number
        $user->sendMessage($courier, 'Test');
    }
}

```

## Other Handlers 其他处理函数

使用类似的方式, 我们可以改进和解耦很多其他类型的"处理函数". 将这些处理函数限制在_转换层_的状态, 你可以将你庞大的业务逻辑和框架解耦, 并保持整洁的代码结构. 为了巩固这种思想, 我们来看看一个路由过滤器. 该过滤器用来验证当前用户是否是交过钱的_高级_用户套餐. 

```php
Route::filter('premium', function()
{
    return Auth::user() && Auth::user()->plan == 'premium';
});

```

猛一看这路由过滤器没什么问题啊. 这么简单的过滤器能有什么错误? 然而就是是这么小的过滤器, 我们却将我们应用实现的细节暴露了出来. 要注意我们在该过滤器里是写明了要检查 `plan` 变量. 这使得将"套餐方案"在我们应用中的代表值 (译者注: 即 `plan` 变量的值) 暴露在了路由/传输层里面. 现在我们若想调整"高级套餐"在数据库或用户模型的代表值, 我们竟然就需要改这个路由过滤器! 

让我们简单改一点儿: 

```php
Route::filter('premium', function()
{
    return Auth::user() && Auth::user()->isPremium();
});

```

小小的改变就带来巨大的效果, 并且代价也很小. 我们将判断用户是否使用高级套餐的逻辑放在了用户模型里, 这样就从路由过滤器里去掉了对套餐判断的实现细节. 我们的过滤器不再需要知道具体怎么判断用户是不是高级套餐了, 它只要简单的把这个问题交给用户模型. 现在如果我们想调整高级套餐在数据库里的细节, 也不必再去改动路由过滤器了! 

### Who Is Responsible? 谁负责? 
>
> 在这里我们又一次讨论了_责任_的概念. 记住, 始终保持一个类应该有什么样的责任, 应该知道什么. 避免在处理函数这种传输层直接编写太多你应用的业务逻辑. 

译者注: 本文多次出现_transport layer_, _translation layer_, 分别译作传输层和转换层. 其实他们应当指代的同一种东西. 

# Extending The Framework 扩展框架

## Introduction 介绍

为了方便你自定义框架核心组件, Laravel 提供了大量可以扩展的地方. 你甚至可以完全替换掉旧组件. 例如: 哈希器遵守了 `HasherInterface` 接口, 你可以按照你自己应用的需求来重新实现. 你也可以扩展 `Request` 对象, 添加你自己用的顺手的"helper"方法. 你甚至可以添加全新的身份认证, 缓存和会话机制! 

Laravel 组件通常有两种扩展方式: 在 IoC 容器里面绑定新实现, 或者用 `Manager` 类注册一个扩展, 该扩展采用了工厂模式实现. 在本章中我们将探索不同的扩展方式并检查我们都需要些什么代码. 

### Methods Of Extension 扩展方式
>
> 要记住 Laravel 通常有以下两种扩展方式: 通过 IoC 绑定和通过 `Manager` 类 (下文译作"管理类") . 其中管理类实现了工厂设计模式, 负责组件的实例化. 比如缓存和会话机制. 

## Manager & Factories 管理者和工厂

Laravel 有好多 `Manager` 类用来管理基于驱动的组件的生成过程. 基于驱动的组件包括: 缓存, 会话, 身份认证, 队列组件等. 管理类负责根据应用程序的配置, 来生成特定的驱动实例. 比如: `CacheManager` 可以创建 APC, Memcached, Native, 还有其他不同的缓存驱动的实现. 

每个管理类都包含名为 `extend` 的方法, 该方法可用于将新功能注入到管理类中. 下面我们将逐个介绍管理类, 为你展示如何注入自定义的驱动. 

### Learn About Your Managers 如何了解你的管理类
>
> 请花点时间看看 Laravel 中各个 `Manager` 类的代码, 比如 `CacheManager` 和 `SessionManager` . 通过阅读这些代码能让你对 Laravel 的管理类机制更加清楚透彻. 所有的管理类都继承自 `Illuminate\Support\Manager` 基类, 该基类为每一个管理类提供了一些有效且通用的功能. 

## Cache 缓存

要扩展 Laravel 的缓存机制, 我们将使用 `CacheManager` 里的 `extend` 方法来绑定我们自定义的缓存驱动. 扩展其他的管理类也是类似的. 比如, 我们想注册一个新的缓存驱动, 名叫"mongo", 代码可以这样写: 

```php
Cache::extend('mongo', function($app)
{
    // Return Illuminate\Cache\Repository instance...
});

```

 `extend` 方法的第一个参数是你要定义的驱动的名字. 该名字对应着 `app/config/cache.php` 配置文件中的 `driver` 项. 第二个参数是一个匿名函数 (闭包) , 该匿名函数有一个 `$app` 参数是 `Illuminate\Foundation\Application` 的实例也是一个 IoC 容器, 该匿名函数要返回一个 `Illuminate\Cache\Repository` 的实例. 

要创建我们自己的缓存驱动, 首先要实现 `Illuminate\Cache\StoreInterface` 接口. 所以我们用 MongoDB 来实现的缓存驱动就可能看上去是这样: 

```php
class MongoStore implements Illuminate\Cache\StoreInterface {
    public function get($key) {}
    public function put($key, $value, $minutes) {}
    public function increment($key, $value = 1) {}
    public function decrement($key, $value = 1) {}
    public function forever($key, $value) {}
    public function forget($key) {}
    public function flush() {}
}
```

我们只需使用 MongoDB 链接来实现上面的每一个方法即可. 一旦实现完毕, 就可以照下面这样完成该驱动的注册: 

```php
use Illuminate\Cache\Repository;
Cache::extend('mongo', function($app)
{
    return new Repository(new MongoStore);
}

```

你可以像上面的例子那样来创建 `Illuminate\Cache\Repository` 的实例. 也就是说通常你不需要创建你自己的仓库类 (Repository) . 

如果你不知道要把自定义的缓存驱动代码放到哪儿, 可以考虑放到 Packagist 里! 或者你也可以在你应用的主目录下创建一个 `Extensions` 目录. 比如, 你的应用叫做 `Snappy` , 你可以将缓存扩展代码放到 `app/Snappy/Extensions/MongoStore.php` . 不过请记住 Laravel 没有对应用程序的结构做硬性规定, 所以你可以按任意你喜欢的方式组织你的代码. 

### Where To Extend 在哪儿调用 Extend 方法? 
>
> 如果你还发愁在哪儿放注册代码, 先考虑放到服务提供者里吧. 我们之前就讲过, 使用服务提供者是一种非常棒的管理你应用代码的途径. 

## Session 会话

扩展 Laravel 的会话机制和上文的缓存机制一样简单. 和刚才一样, 我们使用 `extend` 方法来注册自定义的代码: 

```php
Session::extend('mongo', function($app)
{
// Return implementation of SessionHandlerInterface
});

```

注意我们自定义的会话驱动 (译者注: 原文是 cache driver, 应该是笔误. 正确应为 session driver) 实现的是 `SessionHandlerInterface` 接口. 这个接口在 PHP 5.4 以上版本才有. 但如果你用的是 PHP 5.3 也别担心, Laravel 会自动帮你定义这个接口的. 该接口要实现的方法不多也不难. 我们用 MongoDB 来实现就像下面这样: 

```php
class MongoHandler implements SessionHandlerInterface {
    public function open($savePath, $sessionName) {}
    public function close() {}
    public function read($sessionId) {}
    public function write($sessionId, $data) {}
    public function destroy($sessionId) {}
    public function gc($lifetime) {}
}

```

这些方法不像刚才的 `StoreInterface` 接口定义的那么容易理解. 我们来挨个简单讲讲这些方法都是干啥的: 

- `open` 方法一般在基于文件的会话系统中才会用到. Laravel 已经自带了一个 `native` 的会话驱动, 使用的就是 PHP 自带的基于文件的会话系统, 你可能永远也不需要在这个方法里写东西. 所以留空就好. 另外这也是一个接口设计的反面教材 (稍后我们会继续讨论这一点) . 
- `close` 方法和 `open` 方法通常都不是必需的. 对大部分驱动来说都不必要实现. 
- `read` 方法应该根据 `$sessionId` 参数来返回对应的会话数据的字符串形式. 在你的会话驱动里, 不论读写都不需要做任何数据序列化工作. 因为 Laravel 会负责数据序列化的. 
- `write` 方法应该将 `$sessionId` 对应的 `$data` 字符串放置在一个持久化存储系统中. 比如 MongoDB, Dynamo 等等. 
- `destroy` 方法应该将 `$sessionId` 对应的数据从持久化存储系统中删除. 
- `gc` 方法应该将所有时间超过参数 `$lifetime` 的数据全都删除, 该参数是一个 UNIX 时间戳. 如果你使用的是类似 Memcached 或 Redis 这种有自主到期功能的存储系统, 那该方法可以留空. 

一旦 `SessionHandlerInterface` 实现完毕, 我们就可以将其注册进会话管理器: 

```php
Session::extend('mongo', function($app)
{
    return new MongoHandler;
});

```

注册完毕后, 我们就可以在 `app/config/session.php` 配置文件里使用 `mongo` 驱动了. 

### Share Your Knowledge 分享你的知识
>
> 你要是写了个自定义的会话处理器, 别忘了在 Packagist 上分享啊! 

## Authentication 身份认证


身份认证模块的扩展方式和缓存与会话的扩展方式一样: 使用我们熟悉的 `extend` 方法就可以进行扩展: 

```php
Auth::extend('riak', function($app)
{
// Return implementation of Illuminate\Auth\UserProviderInterface
});

```

接口 `UserProviderInterface` 负责从各种持久化存储系统——如 MySQL, Riak 等——中获取数据, 然后得到接口 `UserInterface` 的实现对象. 有了这两个接口, Laravel 的身份认证机制就可以不用管用户数据是如何储存的, 究竟哪个类来代表用户对象这种事儿, 从而继续专注于身份认证本身的实现. 

咱们来看一看 `UserProviderInterface` 接口的代码: 

```php
interface UserProviderInterface {
    public function retrieveById($identifier);
    public function retrieveByCredentials(array $credentials);
    public function validateCredentials(UserInterface $user, array $credentials);
}

```


方法 `retrieveById` 通常接受一个数字参数用来表示一个用户, 比如 MySQL 数据库的自增 ID. 该方法要找到匹配该 ID 的 `UserInterface` 的实现对象, 并且将该对象返回. 

 `retrieveByCredentials` 方法接受一个参数作为登录帐号. 该参数是在尝试登录系统时从 `Auth::attempt` 方法传来的. 那么该方法应该"查询"底层的持久化存储系统, 来找到那些匹配到该帐号的用户. 通常该方法会执行一个带有"where"条件的查询来匹配参数里的 `$credentials['username']` . **该方法不应该做任何密码验证. **

 `validateCredentials` 方法会通过比较 `$user` 参数和 `$credentials` 参数来检测用户是否通过认证. 比如, 该方法会调用 `$user->getAuthPassword();` 方法, 将得到的字符串与 `$credentials['password']` 经过 `Hash::make` 处理后的结果进行比对. 

现在我们探索了 `UserProviderInterface` 接口的每一个方法, 接下来咱们看一看 `UserInterface` 接口. 别忘了 `UserInterface` 的实例应当是 `retrieveById` 和 `retrieveByCredentials` 方法的返回值: 

```php
interface UserInterface {
    public function getAuthIdentifier();
    public function getAuthPassword();
}

```

这个接口很简单. `getAuthIdentifier` 方法应当返回用户的"主键". 就像刚才提到的, 在 MySQL 中可能就是自增主键了. `getAuthPassword` 方法应当返回经过散列处理的用户密码. 有了这个接口, 身份认证系统就可以不用关心用户类到底使用了什么 ORM 或者什么存储方式. Laravel 已经在 `app/models` 目录下, 包含了一个默认的 `User` 类且实现了该接口. 所以你可以参考这个类当例子. 

当我们最后实现了 `UserProviderInterface` 接口后, 我们可以将该扩展注册进 `Auth` 里面: 

```php
Auth::extend('riak', function($app)
{
    return new RiakUserProvider($app['riak.connection']);
});

```

使用 `extend` 方法注册好驱动以后, 你就可以在 `app/config/auth.php` 配置文件里面切换到新的驱动了. 

## IoC Based Extension 使用容器进行扩展

Laravel 框架内几乎所有的服务提供者都会绑定一些对象到 IoC 容器里. 你可以在 `app/config/app.php` 文件里找到服务提供者列表. 如果你有时间的话, 你应该大致过一遍每个服务提供者的源码. 这么做你便可以对每个服务提供者有更深的理解, 明白他们都往框架里加了什么东西, 对应的什么键. 那些键就用来联系着各种各样的服务. 

举个例子, `PaginationServiceProvider` 向容器内绑定了一个 `paginator` 键, 对应着一个 `Illuminate\Pagination\Environment` 的实例. 你可以很容易的通过覆盖容器绑定来扩展重写该类. 比如, 你可以创建一个扩展自 `Environment` 类的子类: 

```php
namespace Snappy\Extensions\Pagination;
class Environment extends \Illuminate\Pagination\Environment {
//
}

```

子类写好以后, 你可以再创建个新的 `SnappyPaginationProvider` 服务提供者来扩展其 `boot` 方法, 在里面覆盖 paginator: 

```php
class SnappyPaginationProvider extends PaginationServiceProvider {
    public function boot()
    {
        App::bind('paginator', function()
        {
            return new Snappy\Extensions\Pagination\Environment;
        }

        parent::boot();
    }
}

```

注意这里我们继承了 `PaginationServiceProvider` , 而非默认的基类 `ServiceProvider` . 扩展的服务提供者编写完毕后, 就可以在 `app/config/app.php` 文件里将 `PaginationServiceProvider` 替换为你刚扩展的那个类了. 

这就是扩展绑定进容器的核心类的一般方法. 基本上每一个核心类都以这种方式绑定进了容器, 都可以被重写. 还是那一句话, 读一遍框架内的服务提供者源码吧. 这有助于你熟悉各种类是怎么绑定进容器的, 都绑定的是哪些键. 这是学习 Laravel 框架到底如何运转的好方法. 

## Request Extension 请求的扩展

由于这玩意儿是框架里面非常基础的部分, 并且在请求流程中很早就被实例化, 所以要扩展 `Request` 类的方法与之前相比是有些许不同的. 

首先还是要写个子类: 

```php
namespace QuickBill\Extensions;
class Request extends \Illuminate\Http\Request {
// Custom, helpful methods here...
}

```

子类写好后, 打开 `bootstrap/start.php` 文件. 该文件是应用的请求流程中最早被载入的几个文件之一. 要注意被执行的第一个动作是创建 Laravel 的 `$app` 实例: 

```php
$app = new \Illuminate\Foundation\Application;

```

当新的应用实例创建后, 它将会创建一个 `Illuminate\Http\Request` 的实例并且将其绑定到 IoC 容器里, 键名为 `request` . 所以我们需要找个方法来将一个自定义的类指定为"默认的"请求类, 对不对? 而且幸运的是, 应用实例有一个名为 `requestClass` 的方法就是用来干这事儿的! 所以我们只需要在 `bootstrap/start.php` 文件最上面加一行: 

```php
use Illuminate\Foundation\Application;
Application::requestClass('QuickBill\Extensions\Request');

```

一旦你指定了自定义的请求类, Laravel 将在任何时候都可以使用这个 `Request` 类的实例. 并使你很方便的能随时访问到它, 甚至单元测试也不例外! 

# Single Responsibility Principle 单一职责原则

## Introduction 介绍


罗伯特"鲍勃叔叔"马丁阐述了名为"坚实"的一些设计原则 (译者注: 看下面五个原则的首字母正是 SOLID) . 这些都是制作完善的程序设计的优秀基础, 一共有五个原则: 

- The Single Responsibility Principle 单一职责原则
- The Open Closed Principle 开放封闭原则
- The Liskov Substitution Principle 里氏替换原则
- The Interface Segregation Principle 接口隔离原则
- The Dependency Inversion Principle 依赖反转原则

让我们深入探索一下, 再看点代码样例来说明各个原则. 我们将看到, 每个原则之间都有联系. 如果其中一个原则没有被遵循, 那么其他大部分 (可能不会是全部) 的原则也会出问题. 

## In Action 实践

单一职责原则规定一个类有且仅有一个理由使其改变. 换句话说, 一个类的功能边界和职责应当是十分狭窄且集中的. 我们之前就提到过, 在类的职责问题上, 无知是福. 一个类应当做它该做的事儿, 并且不应当被它的依赖的任何变化所影响到. 

考虑下列类: 

```php
class OrderProcessor {
    public function __construct(BillerInterface $biller)
    {
        $this->biller = $biller;
    }
    public function process(Order $order)
    {
        $recent = $this->getRecentOrderCount($order);
        if($recent > 0)
        {
            throw new Exception('Duplicate order likely.');
        }

        $this->biller->bill($order->account->id, $order->amount);

        DB::table('orders')->insert(array(
            'account'    =>    $order->account->id, 
            'amount'    =>    $order->amount, 
            'created_at'=>    Carbon::now()
        ));
    }
    protected function getRecentOrderCount(Order $order)
    {
        $timestamp = Carbon::now()->subMinutes(5);
        return DB::table('orders')->where('account', $order->account->id)
                                                ->where('created_at', '>=', $timestamps)
                                                ->count();
    }
}
```

上面这个类的职责是什么? 很显然顾名思义, 它是用来处理订单的. 不过由于 `getRecentOrderCount` 这个方法的存在, 这个类就有了在数据库中审查某帐号订单历史来看有没有重复订单的职责. 这个额外的验证职责意味着当我们的存储方式改变或当订单验证规则改变时, 我们的这个订单处理器也要跟着改变. 

我们必须将这个职责抽离出来放到另外的类里面, 比如放到 `OrderRepository` : 

```php
class OrderRepository {
    public function getRecentOrderCount(Account $account)
    {
        $timestamp = Carbon::now()->subMinutes(5);
        return DB::table('orders')->where('account', $account->id)
                                                ->where('created_at', '>=', $timestamp)
                                                ->count();
    }

    public function logOrder(Order $order)
    {
        DB::table('orders')->insert(array(
            'account'    =>    $order->account->id, 
            'amount'    =>    $order->amount, 
            'created_at'=>    Carbon::now()
        ));
    }
}

```

然后我们可以将我们的资料库 (译者注: OrderRepository ) 注入到 `OrderProcessor` 里, 帮后者承担起对账户订单历史的处理责任: 

```php
class OrderProcessor {
    public function __construct(BillerInterface $biller, OrderRepository $orders)
    {
        $this->biller = $biller;
        $this->orders = $orders;
    }

    public function process(Order $order)
    {
        $recent = $this->orders->getRecentOrderCount($order->account);

        if($recent > 0)
        {
            throw new Exception('Duplicate order likely.');
        }

        $this->biller->bill($order->account->id, $order->amount);

        $this->orders->logOrder($order);
    }
}
```

现在我们提取出了收集订单数据的责任, 当读取和写入订单的方式改变时, 我们不再需要修改 `OrderProcessor` 这个类了. 我们的类的职责更加的专注和精确, 这提供了一个更干净, 更有表现力的代码, 同时也是更容易维护的代码. 

请记住, 单一职责原则的关键不仅仅是让函数变短, 而是写出职责更精确更高内聚的类, 所以要确保类里面所有的方法都属于该类的职责之下的. 在建立一个小巧, 清晰且职责明确的类库以后, 我们的代码会更加解耦, 更容易测试, 并且更易于更改. 

# Open Closed Principle 开放封闭原则

## Introduction 介绍

在一个应用的生命周期里, 大部分时间都花在了向现有代码库增加功能, 而非一直从零开始写新功能. 正像你所想的那样, 这会是一个繁琐且令人痛苦的过程. 当你修改代码的时候, 你可能引入新的程序错误, 或者将原来管用的功能搞坏掉. 理想情况下, 我们应该可以像写全新的代码一样, 来快速且简单的修改现有的代码. 只要采用开放封闭原则来正确的设计我们的应用程序, 那么这是可以做到的! 

### Open Closed Principle 开放封闭原则
>
> 开放封闭原则规定代码对扩展是开放的, 对修改是封闭的. 

## In Action 实践

为了演示开放封闭原则, 我们来继续编写上一章节的 `OrderProcecssor` . 考虑下面的 `process` 方法: 

```php
$recent = $this->orders->getRecentOrderCount($order->account);

if($recent > 0)
{
    throw new Exception('Duplicate order likely.');
}

```

这段代码可读性很高, 且因为我们使用了依赖注入, 变得很容易测试. 然而, 如果我们判断订单的规则改变了呢? 如果我们又有新的规则了呢? 更进一步, 如果随着我们的业务发展, 要增加_一大堆_新规则呢? 那我们的 `process` 方法会很快变成一坨难以维护的浆糊. 因为这段代码必须随着每次业务逻辑的改变而跟着改变, 它对修改是开放的, 这违反了开放封闭原则. 记住, 我们希望代码对_扩展_开放, 而不是修改. 

不必再把订单验证直接写在 `process` 方法里面, 我们来定义一个新的接口: `OrderValidator` : 

```php
interface OrderValidatorInterface {
    public function validate(Order $order);
}

```

下一步我们来定义一个实现接口的类, 来预防重复订单: 

```php
class RecentOrderValidator implements OrderValidatorInterface {
    public function __construct(OrderRepository $orders)
    {
        $this->orders = $orders;
    }
    public function validate(Order $order)
    {
        $recent = $this->orders->getRecentOrderCount($order->account);
        if($recent > 0)
        {
            throw new Exception('Duplicate order likely.');
        }
    }
}
```

很好! 我们封装了一个小巧的, 可测试的单一业务逻辑. 咱们来再创建一个来验证账号是否停用吧: 

```php
class SuspendedAccountValidator implements OrderValidatorInterface {
    public function validate(Order $order)
    {
        if($order->account->isSuspended())
        {
            throw new Exception("Suspended accounts may not order.");
        }
    }
}
```

现在我们有两个不同的类实现了 `OrderValidatorInterface` 接口. 咱们将在 `OrderProcessor` 里面使用它们. 我们只需简单的将一个验证器数组注入进订单处理器实例中. 这将使我们以后修改代码时能轻松的添加和删除验证器规则. 

```php
class OrderProcessor {
    public function __construct(BillerInterface $biller, OrderRepository $orders, array $validators = array())
    {
        $this->biller = $bller;
        $this->orders = $orders;
        $this->validators = $validators;
    }
}
```

然后我们只要在 `process` 方法里面循环这个验证器数组即可: 

```php
public function process(Order $order)
{
    foreach($this->validators as $validator)
    {
        $validator->validate($order);
    }

    // Process valid order...
}
```

最后我们在 IoC 容器里面注册 `OrderProcessor` 类: 

```php
App::bind('OrderProcessor', function()
{
    return new OrderProcessor(
        App::make('BillerInterface'), 
        App::make('OrderRepository'), 
        array(
            App::make('RecentOrderValidator'), 
            App::make('SuspendedAccountValidator')
        )
    );
});

```

在现有代码里付出些小努力, 做一些小改动之后, 我们现在可以添加删除新的验证规则而不必修改任何一行现有代码了. 每一个新的验证规则就是对 `OrderValidatorInterface` 的一个实现类, 然后注册进 IoC 容器里. 不必再为那个又大又笨的 `process` 方法做单元测试了, 我们现在可以单独测试每一个验证规则. 现在, 我们的代码对扩展是_开放_的, 对修改是_封闭_的. 

### Leaky Abstractions 抽象的漏洞
>
> 小心那些缺少实现细节的依赖 (译者注: 比如上面的 RecentOrderValidator) . 当一个依赖的实现需要改变时, 不应该要求它的调用者做任何修改. 当需要调用者进行修改时, 这就意味着该依赖_遗漏_了一些实现的细节. 当你的抽象有漏洞的话, 开放封闭原则就不管用了. 

在我们继续学习前, 要记住这些原则不是法律. 这不是说你应用中每一块代码都应该是"热插拔"式的. 例如, 一个仅仅从 MySQL 检索几条记录的小应用程序, 不值得去严格遵守每一条你想到的设计原则. 不要盲目的应用设计原则, 那样你会造出一个"过度设计"的繁琐的系统. 记住这些设计原则是用来解决通用的架构问题, 制造大型容错能力强的应用. 我就这么一说, 你可别把它当作懒惰的借口! 

# Liskov Substitution Principle 里氏替换原则

## Introduction 介绍

别担心, 里氏替换原则读起来吓人学起来简单. 该原则要求: 一个抽象的任意一个实现, 可以被用在任何需要该抽象的地方. 读起来绕口, 用普通人的话来解释一下. 该原则规定: 如果某处代码使用了一个接口的一个实现类, 那么在这里也可以直接使用该接口的任何其他实现类, 不用做出任何修改. 

### Liskov Substitution Principle 里氏替换原则
>
> 该原则规定对象应该可以被该对象子类的实例所替换, 并且不会影响到程序的正确性. 

## In Action 实践

为了说明该原则, 我们继续编写上一章节的 `OrderProcessor` . 看下面的方法: 

```php
public function process(Order $order)
{
    // Validate order...
    $this->orders->logOrder($order);
}

```

注意当我们的 `Order` 通过了验证, 就被 `OrderRepositoryInterface` 的实现对象存储起来了. 假设当我们的业务刚起步时, 我们将订单存储在 CSV 格式的文件系统中. 我们的 `OrderRepositoryInterface` 的实现类是 `CsvOrderRepository` . 现在, 随着我们订单增多, 我们想用一个关系数据库来存储订单. 那么我们来看看新的订单资料库类该怎么编写吧: 

```php
class DatabaseOrderRepository implements OrderRepositoryInterface {
    protected $connection;
    public function connect($username, $password)
    {
        $this->connection = new DatabaseConnection($username, $password);
    }

    public function logOrder(Order $order)
    {
        $this->connection->run('insert into orders values (?, ?)', array(
            $order->id, $order->amount
        ));
    }
}
```

现在我们来研究如何使用这个实现类: 

```php
public function process(Order $order)
{
    // Validate order...

    if($this->repository instanceof DatabaseOrderRepository)
    {
        $this->repository->connect('root', 'password');
    }
    $this->repository->logOrder($order);
}
```

注意在这段代码中, 我们必须在资料库外部检查 `OrderRepositoryInterface` 的实例对象是不是用数据库实现的. 如果是的话, 则必须先连接数据库. 在很小的应用中这可能不算什么问题, 但如果 `OrderRepositoryInterface` 被几十个类调用呢? 我们可能就要把这段"启动"代码在每一个调用的地方复制一遍又一遍. 这让人非常头疼难以维护, 非常容易出错误. 一旦我们忘了将所有调用的地方进行同步修改, 那程序恐怕就会出问题. 

很明显, 上面的例子没有遵循里氏替换原则. 如果不附加"启动"代码来调用 `connect` 方法, 则这段代码就没法用. 好了, 我们已经找到问题所在, 咱们修好他. 下面就是新的 `DatabaseOrderRepository` : 

```php
class DatabaseOrderRepository implements OrderRepositoryInterface {
    protected $connector;
    public function __construct(DatabaseConnector $connector)
    {
        $this->connector = $connector;
    }
    public function connect()
    {
        return $this->connector->bootConnection();
    }
    public function logOrder(Order $order)
    {
        $connection = $this->connect();
        $connection->run('insert into orders values (?, ?)', array(
            $order->id, $order->amount
        ));
    }
}
```

现在 `DatabaseOrderRepository` 掌管了数据库连接, 我们可以把"启动"代码从 `OrderProcessor` 移除了: 

```php
public function process(Order $order)
{
    // Validate order...

    $this->repository->logOrder($order);
}

```

这样一改, 我们就可以想用 `CsvOrderRepository` 也行, 想用 `DatabaseOrderRepository` 也行, 不用改 `OrderProcessor` 一行代码. 我们的代码终于实现了里氏替换原则! 要注意, 我们讨论过的许多架构概念都和_知识_相关. 具体讲, 知识就是一个类和它所具有的_周边领域_, 比如用来帮助类完成任务的外围代码和依赖. 当你要制作一个容错性强大的应用架构时, 限制类的_知识_是一种常用且重要的手段. 

还要注意如果不遵守里氏替换原则, 那后果可能会影响到我们之前已经讨论过的其他原则. 不遵守里氏替换原则, 那么开放封闭原则一定也会被打破. 因为, 如果调用者必须检查实例属于哪个子类的, 那一旦有个新的子类, 调用者就得做出改变.  (译者注: 这就违背了对修改封闭的原则. ) 

### Watch For Leaks 小心遗漏
>
> 你可能注意到这个原则和上一章节提到的"抽象的漏洞"密切相关. 我们的数据库资料库的抽象漏洞就是没有遵守里氏替换原则的第一迹象. 要留意那些漏洞! 

# Interface Segregation Principle 接口隔离原则

## Introduction 介绍

接口隔离原则规定在实现接口的时候, 不能强迫去实现没有用处的方法. 你是否曾被迫去实现一些接口里你用不到的方法? 如果答案是肯定的, 那你可能创建了一个空方法放在那里. 被迫去实现用不到的函数, 这就是一个违背了接口隔离原则的例子. 

在实际操作中, 该原则要求接口必须粒度很细, 且专注于一个领域. 听起来很耳熟? 记住, 所有五个"坚实"原则都是相关的, 也就是说当打破一个原则时, 你通常肯定打破了其他的原则. 在这里当你违背了接口隔离原则后, 肯定也违背了单一职责原则. 

"臃肿"的接口, 有着很多不是所有的实现类都需要的方法. 与其写这样的接口, 不如将其拆分成多个小巧的接口, 里面的方法都是各自领域所需要的. 这样将臃肿接口拆成小巧, 功能集中的接口后, 我们就可以使用小接口来编码, 而不必为我们不需要的功能买单. 

### Interface Segregation Principle 接口隔离原则
>
> 该原则规定, 一个接口的一个实现类, 不应该去实现那些自己用不到的方法. 如果需要, 那就是接口设计有问题, 违背了接口隔离原则. 

## In Action 实践

为了说明该原则, 我们来思考一个关于会话处理的类库. 实际上我们将要考察 PHP 自己的 `SessionHandlerInterface` . 下面是该接口定义的方法, 他们是从 PHP 5.4 版才开始有的: 

```php
interface SessionHandlerInterface {
    public function close();
    public function destroy($sessionId);
    public function gc($maxLifetime);
    public function open($savePath, $name);
    public function read($sesssionId);
    public function write($sessionId, $sessionData);
}

```

现在我们知道接口里面都是什么方法了, 我们打算用 Memcached 来实现它. Memcached 需要实现这个接口里的所有方法么? 不, 里面一半的方法对于 Memcached 来说都是不需要实现的! 

因为 Memcached 会自动清除存储的过期数据, 我们不需要实现 `gc` 方法. 我们也不需要实现 `open` 和 `close` 方法. 所以我们被迫去写空方法来站着位子. 为了解决在这个问题, 我们来定义一个小巧的专门用来垃圾回收的接口: 

```php
interface GarbageCollectorInterface {
    public function gc($maxLifetime);
}

```

现在我们有了一个小巧的接口, 功能单一而专注. 需要垃圾清理的只用依赖这个接口即可, 而不必去依赖整个会话处理. 

为了更深入理解该原则, 我们用另一个例子来强化理解. 想象我们有一个名为 `Contact` 的 Eloquent 类, 定义成这样: 

```php
class Contact extends Eloquent {
    public function getNameAttribute()
    {
        return $this->attributes['name'];
    }
    public function getEmailAttribute()
    {
        return $this->attributes['email'];
    }
}

```

现在我们再假设我们应用里还有一个叫 `PasswordReminder` 的类来负责给用户发送密码找回邮件. 下面是 `PasswordReminder` 的定义方式的一种: 

```php
class PasswordReminder {
    public function remind(Contact $contact, $view)
    {
        // Send password reminder e-mail...
    }
}
```

你可能注意到了, `PasswordReminder` 依赖着 `Contact` 类, 也就是依赖着 Eloquent ORM. 对于一个密码找回系统来说, 依赖着一个特定的 ORM 实在是没必要, 也是不可取的. 切断对该 ORM 的依赖, 我们就可以自由的改变我们后台存储机制或者说 ORM, 同时不会影响到我们的密码找回组件. 重申一遍, 违背了"坚实"原则的任何一条, 就意味着有个类它_知道的_太多了. 

要切断这种依赖, 我们来创建一个 `RemindableInterface` 接口. 事实上 Laravel 已经有了这个接口, 并且默认由 `User` 模型实现了该接口: 

```php
interface RemindableInterface {
    public function getReminderEmail();
}
```

一旦接口定义好了, 我们就可以在模型上实现它: 

```
<!-- lang:php -->
class Contact extends Eloquent implements RemindableInterface {
    public function getReminderEmail()
    {
        return $this->email;
    }
}

```
最终我们可以在 `PasswordReminder` 里面依赖这样一个小巧且专注的接口了: 

```php
class PasswordReminder {
    public function remind(RemindableInterface $remindable, $view)
    {
        // Send password reminder e-mail...
    }
}
```

通过这小小的改动, 我们已经移除了密码找回组件里不必要的依赖, 并且使它足够灵活能使用任何实现了 `RemindableInterface` 的类或 ORM. 这其实正是 Laravel 的密码找回组件如何保持与数据库 ORM 无关的秘诀! 

### Knowledge Is Power 知识就是力量
>
> 我们再次发现了一个使类知道太多东西的陷阱. 通过小心留意是否让一个类知道了太多, 我们就可以遵守所有的"坚实"原则. 

# Dependency Inversion Principle 依赖反转原则

## Introduction 介绍

在整个"坚实"原则概述的旅途中, 我们到达最后一站了! 最后的原则是依赖反转原则, 它规定高等级的代码不应该依赖 (迁就) 低等级的代码. 首先, 高等级的代码应该依赖 (遵从) 着抽象层, 抽象层就像是"中间人"一样, 负责连接着高等级和低等级的代码. 其次, 抽象定义不应该依赖 (迁就) 着具体实现, 但具体实现应该依赖 (遵从) 着抽象定义. 如果这些东西让你极端困惑, 别担心. 接下来我们会将这两方面统统介绍给你. 

### Dependency Inversion Principle 依赖反转原则
>
> 该原则要求高等级代码不应该迁就低等级代码, 抽象定义不应该迁就具体实现. 

## In Action 实践

如果你已经读过了本书前面几个章节, 你就已经很好掌握了依赖反转原则! 为了说明本原则, 让我们考虑下面这个类: 

```php
class Authenticator {
    public function __construct(DatabaseConnection $db)
    {
        $this->db = $db;
    }
    public function findUser($id)
    {
        return $this->db->exec('select * from users where id = ?', array($id));
    }
    public function authenticate($credentials)
    {
        // Authenticate the user...
    }
}

```

你可能猜到了, `Authenticator` 就是用来查找和验证用户的. 继续研究它的构造函数. 我们发现它使用了类型提示, 要求传入一个 `DatabaseConnection` 对象, 所以该验证类和数据库被紧密的联系在一起. 而且基本上讲, 这个数据库还只能是关系数据库. 从而可知, 我们的高级代码 ( `Authenticator` ) 直接的依赖着低级代码 ( `DatabaseConnection` ) . 

首先我们来谈谈"高级代码"和"低级代码". 低级代码用于实现基本的操作, 比如从磁盘读文件, 操作数据库等. 高级代码用于封装复杂的逻辑, 它们依靠低级代码来达到功能目的, 但不能直接和低级代码耦合在一起. 取而代之的是高级代码应该依赖着低级代码的顶层抽象, 比如接口. 不仅如此, 低级代码_也_应当依赖着抽象. 所以我们来写个 `Authenticator` 可以用的接口: 

```php
interface UserProviderInterface {
    public function find($id);
    public function findByUsername($username);
}

```

接下来我们将该接口注入到 `Authenticator` 里面: 

```php
class Authenticator {
    public function __construct(UserProviderInterface $users, HasherInterface $hash)
    {
        $this->hash = $hash;
        $this->users = $users;
    }
    public function findUser($id)
    {
        return $this->users->find($id);
    }
    public function authenticate($credentials)
    {
        $user = $this->users->findByUsername($credentials['username']);
        return $this->hash->make($credentials['password']) == $user->password;
    }
}

```

做了这些小改动后, `Authenticator` 现在依赖于两个高级抽象: `UserProviderInterface` 和 `HasherInterface` . 我们可以向 `Authenticator` 自由的注入这俩接口的任何实现类. 比如, 如果我们的用户存储在 Redis 里面, 我们只需写一个 `RedisUserProvider` 来实现 `UserProviderInterface` 接口即可. `Authenticator` 不再依赖着具体的低级别的存储操作了. 

此外, 由于我们的低级别代码实现了 `UserProviderInterface` 接口, 则我们说该低级代码依赖着这个接口. 

```php
class RedisUserProvider implements UserProviderInterface {
    public function __construct(RedisConnection $redis)
    {
        $this->redis = $redis;
    }
    public function find($id)
    {
        $this->redis->get('users:'.$id);
    }
    public function findByUsername($username)
    {
        $id = $this->redis->get('user:id:'.$username);
        return $this->find($id);
    }
}

```

### Inverted Thinking 反转的思维
>
> 贯彻这一原则会_反转_好多开发者设计应用的方式. 不再将高级代码直接和低级代码以"自上而下"的方式耦合在一起, 这个原则提出**无论**高级还是低级代码都要依赖于一个高层次的抽象. 

在我们没有_反转_ `Authenticator` 的依赖之前, 它除了使用数据库存储系统别无选择. 如果我们改变了存储系统, `Authenticator` 也需要被修改, 这就违背了开放封闭原则. 我们又一次看到, 这些设计原则通常一荣俱荣一损俱损. 

通过强制让 `Authenticator` 依赖着一个存储抽象层, 我们就可以使用任何实现了 `UserProviderInterface` 接口的存储系统, 且不用对 `Authenticator` 本身做任何修改. 传统的依赖关系链已经被**反转**了, 代码变得更灵活, 更加无惧变化! 

