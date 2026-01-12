# 错误处理

Yii 有一个 [yiisoft/error-handler](https://github.com/yiisoft/error-handler) 包，使错误处理比以前更加愉快。特别是，Yii 错误处理程序提供以下功能：

- 用于捕获未处理错误的 [PSR-15](https://www.php-fig.org/psr/psr-15/) 中间件。
- 用于将某些异常映射到自定义响应的 PSR-15 中间件。
- 生产和调试模式。
- 调试模式显示详细信息、堆栈跟踪，具有深色和浅色主题以及方便的按钮来搜索错误而无需输入。
- 考虑 PHP 设置。
- 处理内存不足错误、致命错误、警告、通知和异常。
- 可以使用任何 [PSR-3](https://www.php-fig.org/psr/psr-3/) 兼容的日志记录器进行错误日志记录。
- 根据请求的 mime 类型检测响应格式。
- 开箱即用地支持使用 HTML、纯文本、JSON、XML 和头进行响应。
- 你可以为额外的类型实现自己的错误渲染。

本指南描述了如何在 [Yii 框架](https://www.yiiframework.com/)中使用错误处理程序，有关独立于 Yii 使用它的信息，请参阅[包描述](https://github.com/yiisoft/error-handler)。

## 使用错误处理程序

错误处理程序由两部分组成。一部分是 `Yiisoft\ErrorHandler\Middleware\ErrorCatcher` 中间件，当注册时，它会捕获在中间件堆栈执行期间可能出现的异常并将它们传递给处理程序。
另一部分是错误处理程序本身 `Yiisoft\ErrorHandler\ErrorHandler`，它捕获在中间件堆栈之外发生的异常和致命错误。处理程序还将警告和通知转换为异常并执行更多方便的操作。

错误处理程序在应用程序本身中注册。通常它发生在 `ApplicationRunner` 中。默认情况下，处理程序配置来自容器。你可以在应用程序配置 `config/web.php` 中配置它，如下所示：

```php
use Psr\Log\LoggerInterface;
use Yiisoft\ErrorHandler\ErrorHandler;
use Yiisoft\ErrorHandler\Renderer\HtmlRenderer;
use Yiisoft\ErrorHandler\ThrowableRendererInterface;

return [
    // ...
    ErrorHandler::class => static function (LoggerInterface $logger, ThrowableRendererInterface $renderer) {
        $errorHandler = new ErrorHandler($logger, $renderer);
        // 将保留内存的大小设置为 512 KB。默认为 256KB。
        $errorHandler->memoryReserveSize(524_288);
        return $errorHandler;
    },
    
    ThrowableRendererInterface::class => static fn () => new HtmlRenderer([
        // 默认为包文件 "templates/production.php"。
        'template' => '/full/path/to/production/template/file',
        // 默认为包文件 "templates/development.php"。
        'verboseTemplate' => '/full/path/to/development/template/file',
        // 要显示的源代码行的最大数量。默认为 19。
        'maxSourceLines' => 20,
        // 要显示的跟踪源代码行的最大数量。默认为 13。
        'maxTraceLines' => 5,
        // 带有占位符（file、line、icon）的跟踪头行将被替换。默认为 `null`。
        'traceHeaderLine' => '<a href="ide://open?file={file}&line={line}">{icon}</a>',
    ]),
    // ...
];
```

如前所述，错误处理程序将所有非致命 PHP 错误转换为可捕获的异常（`Yiisoft\ErrorHandler\Exception\ErrorException`）。这意味着你可以使用以下代码来处理 PHP 错误：

```php
try {
    10 / 0;
} catch (\Yiisoft\ErrorHandler\Exception\ErrorException $e) {
    // 写入日志或执行其他操作。
}
// 执行继续...
```

该包还有另一个中间件 `Yiisoft\ErrorHandler\Middleware\ExceptionResponder`。
此中间件将某些异常映射到自定义响应。在应用程序配置中配置它，如下所示：

```php
use Psr\Http\Message\ResponseFactoryInterface;
use Yiisoft\ErrorHandler\Middleware\ExceptionResponder;
use Yiisoft\Injector\Injector;

return [
    // ...
    ErrorHandler::class => static function (ResponseFactoryInterface $responseFactory, Injector $injector) {
        $exceptionMap = [
            // 工厂创建响应时使用的状态码。
            MyNotFoundException::class => 404,
            // 必须返回 `Psr\Http\Message\ResponseInterface` 的 PHP 可调用对象。
            MyHttpException::class => static fn (MyHttpException $exception) => new MyResponse($exception),
            // ...
        ],
        
        return new ExceptionResponder($exceptionMap, $responseFactory, $injector);
    },
];
```

请注意，在配置应用程序中间件堆栈时，你必须将 `Yiisoft\ErrorHandler\Middleware\ExceptionResponder` 放在 `Yiisoft\ErrorHandler\Middleware\ErrorCatcher` 之前。

## 渲染错误数据

其中一个渲染器可以将错误数据渲染为特定格式。
以下渲染器开箱即用：

- `Yiisoft\ErrorHandler\Renderer\HeaderRenderer` - 将错误渲染为 HTTP 头。用于 `HEAD` 请求。
- `Yiisoft\ErrorHandler\Renderer\HtmlRenderer` - 将错误渲染为 HTML。
- `Yiisoft\ErrorHandler\Renderer\JsonRenderer` - 将错误渲染为 JSON。
- `Yiisoft\ErrorHandler\Renderer\PlainTextRenderer` - 将错误渲染为纯文本。
- `Yiisoft\ErrorHandler\Renderer\XmlRenderer` - 将错误渲染为 XML。

渲染器根据是否启用调试模式生成详细的错误数据。

关闭调试模式的头渲染示例：

```
...
X-Error-Message: An internal server error occurred.
...
```

打开调试模式的头渲染示例：

```
...
X-Error-Type: Error
X-Error-Message: Call to undefined function App\Controller\myFunc()
X-Error-Code: 0
X-Error-File: /var/www/yii/app/src/Controller/SiteController.php
X-Error-Line: 21
...
```

关闭调试模式的 JSON 渲染输出示例：

```json
{"message":"An internal server error occurred."}
```

打开调试模式的 JSON 渲染输出示例：

```json
{
    "type": "Error",
    "message": "Call to undefined function App\\Controller\\myFunc()",
    "code": 0,
    "file": "/var/www/yii/app/src/Controller/SiteController.php",
    "line": 21,
    "trace": [
        {
            "function": "index",
            "class": "App\\Controller\\SiteController",
            "type": "->"
        },
        {
            "file": "/var/www/yii/app/vendor/yiisoft/injector/src/Injector.php",
            "line": 63,
            "function": "invokeArgs",
            "class": "ReflectionFunction",
            "type": "->"
        },
        ...
    ]
}
```

关闭调试模式的 HTML 渲染示例：

![查看生产环境](/images/guide/runtime/view-production.png)

打开调试模式和浅色主题的 HTML 渲染示例：

![查看开发环境浅色主题](/images/guide/runtime/view-development-light.png)

打开调试模式和深色主题的 HTML 渲染示例：

![查看开发环境深色主题](/images/guide/runtime/view-development-dark.png)

错误捕获器根据 `accept` HTTP 头选择如何渲染异常。
如果是 `text/html` 或任何未知的内容类型，它将使用错误或异常 HTML 模板来显示错误。
对于其他 mime 类型，错误处理程序将选择你在错误捕获器中注册的不同渲染器。
默认情况下，它支持 JSON、XML 和纯文本。

### 实现你自己的渲染器

你可以通过在注册错误捕获器中间件时提供自己的 `Yiisoft\ErrorHandler\ThrowableRendererInterface` 实例来自定义错误响应格式。

```php
use Psr\Http\Message\ServerRequestInterface;
use Yiisoft\ErrorHandler\ErrorData;
use Yiisoft\ErrorHandler\ThrowableRendererInterface;

final readonly class MyRenderer implements ThrowableRendererInterface
{
    public function render(Throwable $t, ServerRequestInterface $request = null): ErrorData
    {
        return new ErrorData($t->getMessage());
    }

    public function renderVerbose(Throwable $t, ServerRequestInterface $request = null): ErrorData
    {
        return new ErrorData(
            $t->getMessage(),
            ['X-Custom-Header' => 'value-header'], // 要添加到响应的头。
        );
    }
};
```

你可以在应用程序配置 `config/web.php` 中配置它：

```php
use Psr\Container\ContainerInterface;
use Psr\Http\Message\ResponseFactoryInterface;
use Yiisoft\ErrorHandler\ErrorHandler;
use Yiisoft\ErrorHandler\Middleware\ErrorCatcher;

return [
    // ...
    ErrorCatcher::class => static function (ContainerInterface $container): ErrorCatcher {
        $errorCatcher = new ErrorCatcher(
            $container->get(ResponseFactoryInterface::class),
            $container->get(ErrorHandler::class),
            $container,
        );
        // 返回一个没有指定内容类型渲染器的新实例。
        $errorCatcher = $errorCatcher->withoutRenderers('application/xml', 'text/xml');
        // 返回一个具有指定内容类型和渲染器类的新实例。
        return $errorCatcher->withRenderer('my/format', new MyRenderer());
    },
    // ...
];
```

## 友好异常

Yii 错误渲染器支持[友好异常](https://github.com/yiisoft/friendly-exception)，使你的团队的错误处理体验更加愉快。其思想是提供可读的名称和问题的可能解决方案：

```php
use Yiisoft\FriendlyException\FriendlyExceptionInterface;

final readonly class RequestTimeoutException extends \RuntimeException implements FriendlyExceptionInterface
{
    public function getName(): string
    {
        return 'Request timed out';
    }
    
    public function getSolution(): ?string
    {
        return <<<'SOLUTION'
可能是资源请求未及时响应的结果。尝试增加超时时间。
SOLUTION;
    }
}
```

当应用程序抛出此类异常时，如果调试模式打开，错误渲染器将显示名称和解决方案。
