# 视图注入

视图注入旨在提供一种标准化的方式，将参数传递到应用程序中视图的公共层。它允许开发者管理将在各个视图中可用的数据，确保代码的灵活性和可重用性。

如果你需要使用视图注入，需要安装 `yiisoft/yii-view-renderer` 包：

```sh
composer require yiisoft/yii-view-renderer
```

## 配置

在配置文件 `params.php` 中：


```php
...
'yiisoft/yii-view' => [
        'injections' => [
            Reference::to(ContentViewInjection::class),
            Reference::to(CsrfViewInjection::class),
            Reference::to(LayoutViewInjection::class),
        ],
    ],
```

## 新建注入

首先定义一个实现 `Yiisoft\Yii\View\Renderer\CommonParametersInjectionInterface` 接口的类。该类将负责提供你想要注入到视图模板和布局中的参数。

```php
class MyCustomParametersInjection implements Yiisoft\Yii\View\Renderer\CommonParametersInjectionInterface
{
    // 类属性和方法将在这里定义

    public function __construct(UserService $userService)
    {
        $this->userService = $userService;
    }

    public function getCommonParameters(): array
    {
        return [
            'siteName' => '我的超棒网站',
            'currentYear' => date('Y'),
            'user' => $this->userService->getCurrentUser(),
        ];
    }
}
```

将你的新注入添加到 `params.php` 中：

```php
'yiisoft/yii-view' => [
        'injections' => [
            ...,
            Reference::to(MyCustomParametersInjection::class),
        ],
    ],
```

## 为不同布局使用单独的注入

如果你的应用程序有多个布局，你可以为每个布局创建单独的参数注入。这种方法允许你根据每个布局的特定需求定制注入的参数，从而增强应用程序的灵活性和可维护性。

为特定布局创建自定义 ViewInjection：

```php
readonly final class CartViewInjection implements CommonParametersInjectionInterface
{
    public function __construct(private Cart $cart)
    {
    }

    public function getCommonParameters(): array
    {
        return [
            'cart' => $this->cart,
        ];
    }
}
```

将你的新注入添加到 `params.php` 中的特定布局名称下。在以下示例中，它是 `@layout/cart`：

```php
'yiisoft/yii-view' => [
        'injections' => [
            ...,
            Reference::to(MyCustomParametersInjection::class),
            DynamicReference::to(static function (ContainerInterface $container) {
                $cart = $container
                    ->get(Cart::class);

                return new LayoutSpecificInjections(
                    '@layout/cart', // 用于注入的布局名称

                    new CartViewInjection($cart)
                );
            }),
        ],
    ],
```
