# 依赖注入和容器

## 依赖注入 <span id="dependency-injection"></span>

在面向对象编程中，有两种重用代码的方式：继承和组合。

继承很简单：

```php
class Cache
{
    public function getCachedValue($key)
    {
        // ..
    }
}

final readonly class CachedWidget extends Cache
{
    public function render(): string
    {
        $output = $this->getCachedValue('cachedWidget');
        if ($output !== null) {
            return $output;
        }
        // ...        
    }
}
```

这里的问题是这两者变得不必要地耦合或相互依赖，使它们更加脆弱。

另一种处理方式是组合：

```php
interface CacheInterface
{
    public function getCachedValue($key);
}

final readonly class Cache implements CacheInterface
{
    public function getCachedValue($key)
    {
        // ..
    }
}

final readonly class CachedWidget
{
    public function __construct(
        private CacheInterface $cache
    )
    {
    }
    
    public function render(): string
    {
        $output = $this->cache->getCachedValue('cachedWidget');
        if ($output !== null) {
            return $output;
        }
        // ...        
    }
}
```
我们避免了不必要的继承，并使用接口来减少耦合。你可以在不更改 `CachedWidget` 的情况下替换缓存实现，因此它变得更加稳定。

这里的 `CacheInterface` 是一个依赖项：一个对象依赖的另一个对象。
将依赖项的实例放入对象（`CachedWidget`）的过程称为依赖注入。
有多种执行方式：

- 构造函数注入。最适合强制性依赖项。
- 方法注入。最适合可选依赖项。
- 属性注入。在 PHP 中最好避免，除非可能是数据传输对象。

### 为什么使用私有属性 <span id="why-private-properties"></span>

在上面的组合示例中，请注意 `$cache` 属性被声明为 `private`。

这种方法通过确保对象具有明确定义的交互接口而不是直接属性访问来拥抱组合，使代码更易于维护，并且不太容易出现某些类型的错误。

这种设计选择提供了几个好处：

- **封装**：带有 getter/setter 的私有属性允许你控制访问并在不破坏现有代码的情况下进行未来更改。
- **数据完整性**：Setter 可以在存储值之前验证、规范化或格式化值，确保属性包含有效数据。
- **不可变性**：私有属性支持不可变对象模式，其中 setter `with*()` 方法返回新实例而不是修改当前实例。
- **灵活性**：你可以创建只读或只写属性，或稍后向属性访问添加额外的逻辑。


## DI 容器 <span id="di-container"></span>

注入基本依赖项很简单。你选择一个不关心依赖项的地方，通常是一个动作处理程序，你永远不会对其进行单元测试，创建所需依赖项的实例并将它们传递给依赖类。

当总体依赖项很少且没有嵌套依赖项时，这种方法效果很好。当有很多依赖项且每个依赖项本身都有依赖项时，实例化整个层次结构就变成了一个繁琐的过程，需要大量代码，并可能导致难以调试的错误。

此外，许多依赖项（例如某些第三方 API 包装器）对于使用它的任何类都是相同的。
因此，有必要：

- 定义如何实例化这样的 API 包装器。
- 在需要时实例化它，并且每个请求只实例化一次。

这就是依赖容器的用途。

依赖注入（DI）容器是一个知道如何实例化和配置对象及其所有依赖对象的对象。[Martin Fowler 的文章](https://martinfowler.com/articles/injection.html)很好地解释了为什么 DI 容器有用。在这里，我们将主要解释 Yii 提供的 DI 容器的使用。

Yii 通过 [yiisoft/di](https://github.com/yiisoft/di) 包和 [yiisoft/injector](https://github.com/yiisoft/injector) 包提供 DI 容器功能。

### 配置容器 <span id="configuring-container"></span>

因为要创建一个新对象，你需要它的依赖项，所以应该尽早注册它们。
你可以在应用程序配置 `config/web.php` 中执行此操作。对于以下服务：
You can do it in the application configuration, `config/web.php`. For the following service:

```php
final class MyService implements MyServiceInterface
{
    public function __construct(int $amount)
    {
    }

    public function setDiscount(int $discount): void
    {
    
    }
}
```

配置可以是：

```php
return [
    MyServiceInterface::class => [
        'class' => MyService::class,
        '__construct()' => [42],
        'setDiscount()' => [10],
    ],
];
```

这等同于以下内容：

```php
$myService = new MyService(42);
$myService->setDiscount(10);
```

还有其他声明依赖项的方法：

```php
// 为接口声明一个类，自动解析依赖项
EngineInterface::class => EngineMarkOne::class,

// 数组定义（与上面相同）
'full_definition' => [
    'class' => EngineMarkOne::class,
    '__construct()' => [42],
    '$propertyName' => 'value',
    'setX()' => [42],
],

// 闭包
'closure' => static function(ContainerInterface $container) {
    return new MyClass($container->get('db'));
},

// 静态调用
'static_call' => [MyFactory::class, 'create'],

// 对象实例
'object' => new MyClass(),
    'object' => new MyClass(),
];
```

### 注入依赖项 <span id="injecting-dependencies"></span>

在类中直接引用容器是一个坏主意，因为代码变得不通用，与容器接口耦合，更糟糕的是，依赖项变得隐藏。
因此，Yii 通过根据方法参数类型在某些构造函数和方法中自动注入容器中的对象来反转控制。

这主要在动作处理程序的构造函数和处理方法中完成：

```php
use \Yiisoft\Cache\CacheInterface;

final readonly class MyController
{
    public function __construct(
        private CacheInterface $cache
    )
    {
        $this->cache = $cache;    
    }

    public function actionDashboard(RevenueReport $report)
    {
        $reportData = $this->cache->getOrSet('revenue_report', function() use ($report) {
            return $report->getData();               
        });

        return $this->render('dashboard', [
           'reportData' => $reportData,
        ]);
    }
}
```

由于是 [yiisoft/injector](https://github.com/yiisoft/injector) 实例化并调用动作处理程序，它会检查构造函数和方法参数类型，从容器中获取这些类型的依赖项并将它们作为参数传递。这通常称为自动装配。它也适用于子依赖项，也就是说，如果你没有显式提供依赖项，容器会首先检查它是否有这样的依赖项。
只需声明你需要的依赖项，它就会自动从容器中获取。


## 参考资料 <span id="references"></span>

- [Martin Fowler 的控制反转容器和依赖注入模式](https://martinfowler.com/articles/injection.html)
