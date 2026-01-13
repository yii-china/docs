# 操作

在 Web 应用程序中，请求 URL 决定执行什么。匹配由配置了多个路由的路由器完成。每个路由都可以附加到一个中间件，该中间件在给定请求时生成响应。由于中间件总体上可以链接并可以将实际处理传递给下一个中间件，我们将实际执行工作的中间件称为操作。

有多种方式来描述操作。最简单的方式是使用闭包：

```php
use \Psr\Http\Message\ServerRequestInterface;
use \Psr\Http\Message\ResponseInterface;
use Yiisoft\Router\Route;

Route::get('/')->action(function (ServerRequestInterface $request) use ($responseFactory): ResponseInterface {
    $response = $responseFactory->createResponse();
    $response->getBody()->write('You are at homepage.');
    return $response;
});
```

对于简单的处理来说这很好，因为任何更复杂的处理都需要获取依赖项，因此一个好主意是将处理移到类方法中。可以使用回调中间件来实现此目的：

```php
use Yiisoft\Router\Route;

Route::get('/')->action([FrontPageAction::class, 'run']),
```

类本身如下所示：

```php
use \Psr\Http\Message\ServerRequestInterface;
use \Psr\Http\Message\ResponseInterface;

final readonly class FrontPageAction
{
    public function run(ServerRequestInterface $request): ResponseInterface
    {
        // build response for a front page    
    }
}
```

在许多情况下，将多个路由的处理分组到单个类中是有意义的：


```php
use Yiisoft\Router\Route;

Route::get('/post/index')->action([PostController::class, 'actionIndex']),
Route::get('/post/view/{id:\d+}')->action([PostController::class, 'actionView']),
```

类本身如下所示：

```php
use \Psr\Http\Message\ServerRequestInterface;
use \Psr\Http\Message\ResponseInterface;

final readonly class PostController
{
    public function actionIndex(ServerRequestInterface $request): ResponseInterface
    {
        // render posts list
    }
    
    
    public function actionView(ServerRequestInterface $request): ResponseInterface
    {
        // render a single post      
    }
}
```

我们通常将这样的类称为"控制器"。

## 自动装配

操作类的构造函数和操作方法都会自动从依赖注入容器中获取服务：

```php
use \Psr\Http\Message\ServerRequestInterface;
use \Psr\Http\Message\ResponseInterface;
use Psr\Log\LoggerInterface;

final readonly class PostController
{
    public function __construct(
        private PostRepository $postRepository
    )
    {
    }

    public function actionIndex(ServerRequestInterface $request, LoggerInterface $logger): ResponseInterface
    {
        $logger->debug('Rendering posts list');
        // render posts list
    }
    
    
    public function actionView(ServerRequestInterface $request): ResponseInterface
    {
        // render a single post      
    }
}
```

在上面的示例中，`PostRepository` 通过构造函数自动注入。这意味着它在每个操作中都可用。Logger 仅注入到 `index` 操作中。

