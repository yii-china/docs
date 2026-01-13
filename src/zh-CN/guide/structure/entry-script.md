# 入口脚本

入口脚本是应用程序引导过程的第一步。应用程序（无论是 Web 应用程序还是控制台应用程序）都有一个入口脚本。最终用户向入口脚本发出请求，入口脚本实例化应用程序实例并将请求转发给它们。

Web 应用程序的入口脚本必须存储在 Web 可访问的目录下，以便最终用户可以访问它们。它们通常命名为 `index.php`，但也可以使用任何其他名称，只要 Web 服务器可以定位它们。

控制台应用程序的入口脚本是 `./yii`。

入口脚本在 `ApplicationRunner` 的帮助下主要执行以下工作：

* 注册 [Composer 自动加载器](https://getcomposer.org/doc/01-basic-usage.md#autoloading)；
* 获取配置；
* 使用配置初始化依赖注入容器；
* 获取请求的实例。
* 将其传递给 `Application` 进行处理并从中获取响应。
* 在发射器的帮助下，将响应对象转换为发送到客户端浏览器的实际 HTTP 响应。

## Web 应用程序 <span id="web-applications"></span>

以下是应用程序模板的入口脚本中的代码：

```php
<?php

declare(strict_types=1);

use App\ApplicationRunner;

// PHP built-in server routing.
if (PHP_SAPI === 'cli-server') {
    // Serve static files as is.
    if (is_file(__DIR__ . $_SERVER['REQUEST_URI'])) {
        return false;
    }

    // Explicitly set for URLs with dot.
    $_SERVER['SCRIPT_NAME'] = '/index.php';
}

require_once dirname(__DIR__) . '/vendor/autoload.php';

$runner = new ApplicationRunner();
// Development mode:
$runner->debug();
// Run application:
$runner->run();
```


## 控制台应用程序 <span id="console-applications"></span>

类似地，以下是控制台应用程序的入口脚本代码：

```php
#!/usr/bin/env php
<?php

declare(strict_types=1);

use Psr\Container\ContainerInterface;
use Yiisoft\Config\Config;
use Yiisoft\Di\Container;
use Yiisoft\Di\ContainerConfig;
use Yiisoft\Yii\Console\Application;
use Yiisoft\Yii\Console\Output\ConsoleBufferedOutput;

define('YII_ENV', getenv('env') ?: 'production');

require_once 'vendor/autoload.php';

$config = new Config(
    __DIR__,
    '/config/packages',
);

$containerConfig = ContainerConfig::create()
    ->withDefinitions($config->get('console'))
    ->withProviders($config->get('providers-console'));
$container = new Container($containerConfig);

/** @var ContainerInterface $container */
$container = $container->get(ContainerInterface::class);

$application = $container->get(Application::class);
$exitCode = 1;

try {
    $application->start();
    $exitCode = $application->run(null, new ConsoleBufferedOutput());
} catch (\Error $error) {
    $application->renderThrowable($error, new ConsoleBufferedOutput());
} finally {
    $application->shutdown($exitCode);
    exit($exitCode);
}
```

## 替代运行时

对于诸如 RoadRunner 或 Swoole 之类的替代运行时，应使用特殊的入口脚本。请参阅：

- [使用 Yii 与 RoadRunner](../tutorial/using-yii-with-roadrunner.md)
- [使用 Yii 与 Swoole](../tutorial/using-yii-with-swoole.md)
