# 中间件

Yii 使用围绕 [PSR-7 HTTP 消息接口](https://www.php-fig.org/psr/psr-7/)和 [PSR-15 请求处理器/中间件接口](https://www.php-fig.org/psr/psr-15/)构建的抽象层来处理 HTTP。

应用程序由一个或多个中间件组成。中间件在请求和响应之间运行。当请求 URL 时，请求对象被传递给中间件调度器，该调度器开始依次执行中间件。每个中间件在给定请求时可以：

- 将请求传递给下一个中间件，在其之前/之后执行一些工作。
- 形成响应并返回它。

根据使用的中间件，应用程序行为可能会有很大差异。

![中间件](/images/guide/middleware.svg)

在上图中，每个下一个中间件都包装了前一个中间件。或者，它可以表示如下：

![中间件](/images/guide/middleware_alternative.svg)

## 使用中间件

任何与 [PSR-15](https://www.php-fig.org/psr/psr-15/) 兼容的中间件都可以与 Yii 一起使用，并且有很多这样的中间件。假设您需要向应用程序的某个 URL 添加基本身份验证。依赖于 URL 的中间件使用路由器配置，因此您需要更改路由器工厂。

身份验证中间件由 `middlewares/http-authentication` 包实现，因此在应用程序根目录中执行 `composer require middlewares/http-authentication`。

现在在 DI 容器配置 `config/web.php` 中注册中间件：

```php
<?php
\Middlewares\BasicAuthentication::class => [
    'class' => \Middlewares\BasicAuthentication::class,
    '__construct()' => [
        'users' => [
            'foo' => 'bar',
        ],
    ],
    'realm()' => ['Access to the staging site via basic auth'],
    'attribute()' => ['username'],
],
```

在 `config/routes.php` 中，添加新路由：

```php
<?php

declare(strict_types=1);

use Yiisoft\Router\Route;
use App\Controller\SiteController;
use Middlewares\BasicAuthentication;


return [
    //...
    Route::get('/basic-auth')->([SiteController::class, 'auth'])
        ->name('site/auth')
        ->addMiddleware(BasicAuthentication::class)
];
```

在配置路由时，您将 `/basic-auth` URL 绑定到由基本身份验证和操作本身组成的中间件链。链是一个特殊的中间件，它执行配置的所有中间件。

操作本身可以如下所示：

```php
public function auth(ServerRequestInterface $request): ResponseInterface
{
    $response = $this->responseFactory->createResponse();
    $response->getBody()->write('Hi ' . $request->getAttribute('username'));
    return $response;
}
```

基本身份验证中间件将 `username` 写入请求属性，因此您可以在需要时访问数据。

要将中间件应用于整个应用程序而不考虑 URL，请调整 `config/application.php`：

```php
return [
    Yiisoft\Yii\Http\Application::class => [
        '__construct()' => [
            'dispatcher' => DynamicReference::to(static function (Injector $injector) {
                return ($injector->make(MiddlewareDispatcher::class))
                    ->withMiddlewares(
                        [
                            ErrorCatcher::class,
                            BasicAuthentication::class,
                            SessionMiddleware::class,
                            CsrfMiddleware::class,
                            Router::class,
                        ]
                    );
            }),
            'fallbackHandler' => Reference::to(NotFoundHandler::class),
        ],
    ],
];
```

## 创建自己的中间件

要创建中间件，您需要实现 `Psr\Http\Server\MiddlewareInterface` 的单个 `process` 方法：

```php
public function process(ServerRequestInterface $request, RequestHandlerInterface $next): ResponseInterface;
```

有多种方式来处理请求，选择哪种方式取决于中间件应该实现什么。

### 直接形成响应

要直接响应，需要通过构造函数传递响应工厂：

```php
<?php
declare(strict_types=1);

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

final readonly class RespondingMiddleware implements MiddlewareInterface
{
    public function __construct(
        private ResponseFactoryInterface $responseFactory
    )
    {
    }

    public function process(ServerRequestInterface $request, RequestHandlerInterface $next): ResponseInterface
    {
        $response = $this->responseFactory->createResponse();
        $response->getBody()->write('Hello!');
        return $response;
    }
}
```

### 将处理委托给下一个中间件

如果中间件不打算形成响应或更改请求，或者这次无法执行任何操作，则可以将处理留给堆栈中的下一个中间件：

```php
return $next->handle($request);
```

如果您需要将数据传递给下一个中间件，可以使用请求属性：

```php
$request = $request->withAttribute('answer', 42);
return $next->handle();
``` 

要在下一个中间件中获取它：

```php
$answer = $request->getAttribute('answer');
```

### 捕获响应以操作它

您可能想要捕获响应以操作它。这对于添加 CORS 头、gzip 压缩内容等很有用。

```php
$response = $next->handle($request);
// 额外处理
return $response;
```
