# 日志

Yii 依赖 [PSR-3 接口](https://www.php-fig.org/psr/psr-3/) 进行日志记录，因此你可以配置任何兼容 PSR-3 的日志库来完成实际工作。

Yii 提供了自己的日志记录器，它具有高度的可定制性和可扩展性。
使用它，你可以记录各种类型的消息，对它们进行过滤，
并将它们收集到不同的目标，例如文件或电子邮件。

使用 Yii 日志框架涉及以下步骤：
 
* 在代码的各个位置记录[日志消息](#log-messages)；
* 在应用程序配置中配置[日志目标](#log-targets)以过滤和导出日志消息；
* 检查由不同目标导出的已过滤日志消息（例如 Yii 调试器）。

本节重点介绍前两个步骤。

## 日志消息 <span id="log-messages"></span>

要记录日志消息，你需要一个 PSR-3 日志记录器的实例。
编写日志消息的类应该将其作为依赖项接收：

```php
class MyService
{
    private $logger;
    
    public function __construct(\Psr\Log\LoggerInterface $logger)
    {
        $this->logger = $logger;    
    }
}
```

记录日志消息就像调用以下与日志级别对应的日志记录方法之一一样简单：

- `emergency` - 系统不可用。
- `alert` - 必须立即采取行动。
  示例：整个网站宕机、数据库不可用等。
  这应该触发短信警报并唤醒你。
- `critical` - 严重情况。示例：应用程序组件不可用、意外异常。
- `error` - 不需要立即采取行动但通常应该记录和监控的运行时错误。
- `warning` - 不是错误的异常情况。示例：使用已弃用的 API、API 使用不当、
  不一定错误但不理想的事情。
- `notice` - 正常但重要的事件。
- `info` - 有趣的事件。示例：用户登录、SQL 日志。
- `debug` - 详细的调试信息。

每个方法都有两个参数。
第一个是消息。
第二个是上下文数组，通常包含结构化数据，
这些数据不太适合放在消息中，但仍然提供重要信息。
如果你提供异常作为上下文，应该传递 "exception" 键。
另一个特殊键是 "category"。类别有助于更好地组织和过滤日志消息。

```php
use \Psr\Log\LoggerInterface;

final readonly class MyService
{
    public function __construct(
        private LoggerInterface $logger
    )
    {    
    }

    public function serve(): void
    {
        $this->logger->info('MyService is serving', ['context' => __METHOD__]);    
    }
}
```

在为消息决定类别时，你可以选择分层命名方案，这将使
[日志目标](#log-targets)更容易根据类别过滤消息。一个简单而有效的命名方案
是使用 PHP 魔术常量 `__METHOD__` 作为类别名称。这也是 Yii 框架核心代码中使用的方法。

`__METHOD__` 常量的值是出现该常量的方法名称（带有完全限定的类名前缀）。
例如，如果上述代码行在此方法中调用，它等于字符串 `'App\\Service\\MyService::serve'`。

> [!IMPORTANT]
> 上述日志记录方法实际上是 [[\Psr\Log\LoggerInterface::log()]] 的快捷方式。

请注意，PSR-3 包提供了 `\Psr\Log\NullLogger` 类，它提供相同的方法集但不记录任何内容。
这意味着你不必使用 `if ($logger !== null)` 检查日志记录器是否已配置，而是可以假设日志记录器始终存在。


## 日志目标 <span id="log-targets"></span>

日志目标是扩展 [[\Yiisoft\Log\Target]] 的类的实例。它根据严重级别和类别过滤日志消息，
然后将它们导出到某个媒介。例如，
[[\Yiisoft\Log\Target\File\FileTarget|文件目标]]将过滤后的日志消息导出到文件，
而 [[Yiisoft\Log\Target\Email\EmailTarget|电子邮件目标]] 将日志消息导出到指定的电子邮件地址。

你可以通过 `\Yiisoft\Log\Logger` 构造函数配置它们，在应用程序中注册多个日志目标：

```php
use \Psr\Log\LogLevel;

$fileTarget = new \Yiisoft\Log\Target\File\FileTarget('/path/to/app.log');
$fileTarget->setLevels([LogLevel::ERROR, LogLevel::WARNING]);

$emailTarget = new \Yiisoft\Log\Target\Email\EmailTarget($mailer, ['to' => 'log@example.com']);
$emailTarget->setLevels([LogLevel::EMERGENCY, LogLevel::ALERT, LogLevel::CRITICAL]);
$emailTarget->setCategories(['Yiisoft\Cache\*']); 

$logger = new \Yiisoft\Log\Logger([
    $fileTarget,
    $emailTarget
]);
```

在上面的代码中，注册了两个日志目标：

* 第一个目标选择错误和警告消息并将它们写入 `/path/to/app.log` 文件；
* 第二个目标选择名称以 `Yiisoft\Cache\` 开头的类别下的紧急、警报和严重消息，
并将它们通过电子邮件发送到 `admin@example.com` 和 `developer@example.com`。

Yii 附带以下内置日志目标。请参阅有关这些类的 API 文档以了解如何配置和使用它们。

* [[\Yiisoft\Log\PsrTarget]]：将日志消息传递给另一个兼容 PSR-3 的日志记录器。
* [[\Yiisoft\Log\StreamTarget]]：将日志消息写入指定的输出流。
* [[\Yiisoft\Log\Target\Db\DbTarget]]：将日志消息保存在数据库中。
* [[\Yiisoft\Log\Target\Email\EmailTarget]]：将日志消息发送到预先指定的电子邮件地址。
* [[\Yiisoft\Log\Target\File\FileTarget]]：将日志消息保存在文件中。
* [[\Yiisoft\Log\Target\Syslog\SyslogTarget]]：通过调用 PHP 函数 `syslog()` 将日志消息保存到系统日志。

下面，我们将描述所有日志目标共有的功能。


### 消息过滤 <span id="message-filtering"></span>

对于每个日志目标，你可以配置其级别和类别，以指定目标应处理哪些严重级别和类别的消息。

目标的 `setLevels()` 方法接受一个由一个或多个 `\Psr\Log\LogLevel` 常量组成的数组。

默认情况下，目标将处理*任何*严重级别的消息。

目标的 `setCategories()` 方法接受一个由消息类别名称或模式组成的数组。
目标将仅处理其类别可以在此数组中找到或匹配其中一个模式的消息。
类别模式是在其末尾带有星号 `*` 的类别名称前缀。如果类别名称以模式的相同前缀开头，
则它匹配类别模式。例如，`Yiisoft\Cache\Cache::set` 和 `Yiisoft\Cache\Cache::get`
都匹配模式 `Yiisoft\Cache\*`。

默认情况下，目标将处理*任何*类别的消息。

除了通过 `setCategories()` 方法允许类别外，你还可以通过 `setExcept()` 方法拒绝某些类别。
如果消息的类别在此属性中找到或匹配其中一个模式，目标将不会处理它。
 
以下目标配置指定目标应仅处理名称匹配 `Yiisoft\Cache\*` 或 `App\Exceptions\HttpException:*`
的类别下的错误和警告消息，但不包括 `App\Exceptions\HttpException:404`。

```php
$fileTarget = new \Yiisoft\Log\Target\File\FileTarget('/path/to/app.log');
$fileTarget->setLevels([LogLevel::ERROR, LogLevel::WARNING]);
$fileTarget->setCategories(['Yiisoft\Cache\*', 'App\Exceptions\HttpException:*']);
$fileTarget->setExcept(['App\Exceptions\HttpException:404']);
```

### 消息格式化 <span id="message-formatting"></span>

日志目标以特定格式导出过滤后的日志消息。
例如，如果你安装了 [[\Yiisoft\Log\Target\File\FileTarget]] 类的日志目标，
你可能会在日志文件中找到类似以下内容的日志消息：

```
2020-12-05 09:27:52.223800 [info][application] Some message

Message context:

time: 1607160472.2238
memory: 4398536
category: 'application'
```

默认情况下，日志消息具有以下格式：

```
Timestamp Prefix[Level][Category] Message Context
```

你可以通过调用 [[\Yiisoft\Log\Target::setFormat()|setFormat()]] 方法自定义此格式，
该方法接受一个返回自定义格式消息的 PHP 可调用对象。

```php
$fileTarget = new \Yiisoft\Log\Target\File\FileTarget('/path/to/app.log');

$fileTarget->setFormat(static function (\Yiisoft\Log\Message $message) {
    $category = strtoupper($message->context('category'));
    return "({$category}) [{$message->level()}] {$message->message()}";
});

$logger = new \Yiisoft\Log\Logger([$fileTarget]);
$logger->info('Text message', ['category' => 'app']);

// Result:
// (APP) [info] Text message
```

此外，如果你对默认消息格式满意，但需要更改时间戳格式或向消息添加自定义数据，
你可以调用 [[\Yiisoft\Log\Target::setTimestampFormat()|setTimestampFormat()]]
和 [[\Yiisoft\Log\Target::setPrefix()|setPrefix()]] 方法。例如，以下代码更改时间戳格式
并配置日志目标以当前用户 ID 为每条日志消息添加前缀（出于隐私原因删除了 IP 地址和会话 ID）。

```php
$fileTarget = new \Yiisoft\Log\Target\File\FileTarget('/path/to/app.log');
$userId = '123e4567-e89b-12d3-a456-426655440000';

// Default: 'Y-m-d H: i: s.u'
$fileTarget->setTimestampFormat('D d F Y');
// Default: ''
$fileTarget->setPrefix(static fn () => "[{$userId}]");

$logger = new \Yiisoft\Log\Logger([$fileTarget]);
$logger->info('Text', ['category' => 'user']);

// Result:
// Fri 04 December 2020 [123e4567-e89b-12d3-a456-426655440000][info][user] Text
// Message context: ...
// Common context: ...
```

传递给 [[\Yiisoft\Log\Target::setFormat()|setFormat()]]
和 [[\Yiisoft\Log\Target::setPrefix()|setPrefix()]] 方法的 PHP 可调用对象具有以下签名：

```php
function (\Yiisoft\Log\Message $message, array $commonContext): string;
```

除了消息前缀外，日志目标还会将一些公共上下文信息附加到每条日志消息。
你可以通过调用目标的 [[\Yiisoft\Log\Target::setCommonContext()|setCommonContext()]]
方法来调整此行为，传递一个你想要包含的 `key => value` 格式的数据数组。
例如，以下日志目标配置指定仅将 `$_SERVER` 变量的值附加到日志消息。

```php
$fileTarget = new \Yiisoft\Log\Target\File\FileTarget('/path/to/app.log');
$fileTarget->setCommonContext(['server' => $_SERVER]);
```


### 消息跟踪级别 <span id="trace-level"></span>

在开发过程中，通常希望查看每条日志消息的来源。
你可以通过调用 [[\Yiisoft\Log\Logger::setTraceLevel()|setTraceLevel()]] 方法来实现这一点，如下所示：

```php
$logger = new \Yiisoft\Log\Logger($targets);
$logger->setTraceLevel(3);
```

此应用程序配置将跟踪级别设置为 3，因此每条日志消息将附加最多三层记录日志消息的调用堆栈。
你还可以通过调用 [[\Yiisoft\Log\Logger::setExcludedTracePaths()|setExcludedTracePaths()]] 方法
设置要从跟踪中排除的路径列表。

```php
$logger = new \Yiisoft\Log\Logger($targets);
$logger->setExcludedTracePaths(['/path/to/file', '/path/to/folder']);
```

> [!IMPORTANT]
> 获取调用堆栈信息并非易事。因此，你应该仅在开发期间或调试应用程序时使用此功能。


### 消息刷新和导出 <span id="flushing-exporting"></span>

如前所述，日志消息由 [[\Yiisoft\Log\Logger|日志记录器对象]] 在数组中维护。为了限制此数组的内存消耗，
每当数组累积一定数量的日志消息时，日志记录器将把记录的消息刷新到[日志目标](#log-targets)。
你可以通过调用 [[\Yiisoft\Log\Logger::setFlushInterval()]] 方法自定义此数量：


```php
$logger = new \Yiisoft\Log\Logger($targets);
$logger->setFlushInterval(100); // default is 1000
```

> [!IMPORTANT]
> 消息刷新也会在应用程序结束时发生，这确保日志目标可以接收完整的日志消息。

当 [[\Yiisoft\Log\Logger|日志记录器对象]] 将日志消息刷新到[日志目标](#log-targets)时，
它们不会立即导出。相反，仅当日志目标累积一定数量的过滤消息时才会发生消息导出。
你可以通过调用各个[日志目标](#log-targets)的
[[\Yiisoft\Log\Target::setExportInterval()|setExportInterval()]] 方法自定义此数量，如下所示：

```php
$fileTarget = new \Yiisoft\Log\Target\File\FileTarget('/path/to/app.log');
$fileTarget->setExportInterval(100); // default is 1000
```

由于刷新和导出级别设置，默认情况下，当你调用任何日志记录方法时，
你不会立即在日志目标中看到日志消息。这对于某些长时间运行的控制台应用程序可能是个问题。
要使每条日志消息立即出现在日志目标中，你应该将刷新间隔和导出间隔都设置为 1，如下所示：

```php
$fileTarget = new \Yiisoft\Log\Target\File\FileTarget('/path/to/app.log');
$fileTarget->setExportInterval(1);

$logger = new \Yiisoft\Log\Logger([$fileTarget]);
$logger->setFlushInterval(1);
```

> [!NOTE]
> 频繁的消息刷新和导出会降低应用程序的性能。


### 切换日志目标 <span id="toggling-log-targets"></span>

你可以通过调用其 [[\Yiisoft\Log\Target::enable()|enable()]]
和 [[\Yiisoft\Log\Target::disable()|disable()]] 方法来启用或禁用日志目标。
你可以通过日志目标配置或在代码中使用以下 PHP 语句来执行此操作：

```php
$fileTarget = new \Yiisoft\Log\Target\File\FileTarget('/path/to/app.log');
$logger = new \Yiisoft\Log\Logger([$fileTarget, /*Other targets*/]);

foreach ($logger->getTargets() as $target) {
    if ($target instanceof \Yiisoft\Log\Target\File\FileTarget) {
        $target->disable();
    }
}
```

要检查日志目标是否已启用，请调用 `isEnabled()` 方法。
你还可以将可调用对象传递给 [[\Yiisoft\Log\Target::setEnabled()|setEnabled()]]
以定义日志目标是否应启用的动态条件。


### 创建新目标 <span id="new-targets"></span>

创建新的日志目标类很简单。你主要需要实现 [[\Yii\Log\Target::export()]]
抽象方法，该方法将所有累积的日志消息发送到指定的媒介。

以下受保护的方法也可用于子目标：

- `getMessages` - 获取日志消息列表（[[\Yii\Log\Message]] 实例）。
- `getFormattedMessages` - 获取格式化为字符串的日志消息列表。
- `formatMessages` - 获取格式化为字符串的所有日志消息。
- `getCommonContext` - 获取包含 `key => value` 格式的公共上下文数据的数组。

有关更多详细信息，你可以参考包中包含的任何日志目标类。

> [!TIP]
> 你可以尝试使用任何兼容 PSR-3 的日志记录器（例如 [Monolog](https://github.com/Seldaek/monolog)），
通过使用 [[\Yii\Log\PsrTarget]] 而不是创建自己的日志记录器。

```php
/**
 * @var \Psr\Log\LoggerInterface $psrLogger
 */

$psrTarget = new \Yiisoft\Log\PsrTarget($psrLogger);
$logger = new \Yiisoft\Log\Logger([$psrTarget]);

$logger->info('Text message');
```
