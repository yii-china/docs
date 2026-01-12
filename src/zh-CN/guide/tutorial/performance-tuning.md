# 性能调优

影响应用性能的因素有很多。有些是环境因素，有些与您的代码相关，还有一些与 Yii 本身相关。在本节中，我们将列举大部分这些因素，并解释如何通过调整这些因素来提高应用性能。

## 优化 PHP 环境 <span id="optimizing-php"></span>

良好配置的 PHP 环境很重要。要获得最佳性能：

- 使用最新的稳定 PHP 版本。PHP 的主要版本可能会带来显著的性能改进。
- 使用 [Opcache](https://secure.php.net/opcache) 启用字节码缓存。
  字节码缓存避免了为每个传入请求解析和包含 PHP 脚本所花费的时间。
- [调优 `realpath()` 缓存](https://github.com/samdark/realpath_cache_tuner)。
- 确保生产环境中没有安装 [XDebug](https://xdebug.org/)。
- 尝试 [PHP 7 预加载](https://wiki.php.net/rfc/preload)。

## 优化代码 <span id="optimizing-code"></span>

除了环境配置之外，还有一些代码级别的优化可以提高应用的性能：

- 注意[算法复杂度](https://en.wikipedia.org/wiki/Time_complexity)。
  特别要注意 `foreach` 嵌套在 `foreach` 循环中的情况，但也要注意在循环中使用[重量级 PHP 函数](https://stackoverflow.com/questions/2473989/list-of-big-o-for-php-functions)。
- [加速 array_merge()](https://www.exakat.io/speeding-up-array_merge/)
- [将 foreach() 移到方法内部](https://www.exakat.io/move-that-foreach-inside-the-method/)
- [数组、类和匿名类的内存使用](https://www.exakat.io/array-classes-and-anonymous-memory-usage/)
- 使用带前导反斜杠的完全限定函数名来优化 opcache 性能。
  当从命名空间内调用[某些全局函数](https://github.com/php/php-src/blob/944b6b6bbd6f05ad905f5f4ad07445792bee4027/Zend/zend_compile.c#L4291-L4353)时，PHP 首先在当前命名空间中搜索，然后才回退到全局命名空间。
  添加前导反斜杠（例如，使用 `\count()` 而不是 `count()`）告诉 PHP 直接使用全局函数，避免命名空间查找并提高 opcache 效率。这种优化最好使用 [PHP-CS-Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) 等工具配合 `native_function_invocation` 规则自动实现。

只有当相关代码频繁执行时，上述优化才会给您带来显著的性能提升。这通常是大循环或批处理的情况。

## 使用缓存技术 <span id="using-caching-techniques"></span>

您可以使用各种缓存技术来显著提高应用的性能。例如，如果您的应用允许用户输入 Markdown 格式的文本，您可以考虑缓存解析后的 Markdown 内容，以避免在每个请求中重复解析相同的 Markdown 文本。请参阅[缓存](../caching/overview.md)部分以了解 Yii 提供的缓存支持。

## 优化会话存储 <span id="optimizing-session-storage"></span>

默认情况下，会话数据存储在文件中。该实现从打开会话到通过 `$session->close()` 关闭会话或请求结束时锁定文件。当会话文件被锁定时，所有其他试图使用相同会话的请求都会被阻塞。这是在等待初始请求释放会话文件。这对于开发和可能的小型项目来说是可以的。但是当涉及到处理大量并发请求时，最好使用更复杂的存储，例如 Redis。

这可以通过[通过 php.ini 配置 PHP](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-redis-server-as-a-session-handler-for-php-on-ubuntu-14-04) 或[实现 SessionHandlerInterface](https://www.sitepoint.com/saving-php-sessions-in-redis/) 并按如下方式配置会话服务来完成：

```php
\Yiisoft\Session\SessionInterface::class => [
    'class' => \Yiisoft\Session\Session::class,
    '__construct()' => [[], $myCustomSessionHandler],
],
```

## 优化数据库 <span id="optimizing-databases"></span>

执行数据库查询和从数据库获取数据通常是 Web 应用中的主要性能瓶颈。虽然使用[数据缓存](../caching/data.md)技术可以缓解性能影响，但它并不能完全解决问题。当数据库有大量数据且缓存的数据无效时，如果没有适当的数据库和查询设计，获取最新数据可能会非常昂贵。

提高数据库查询性能的一般技术是为需要过滤的表列创建索引。例如，如果您需要通过 `username` 查找用户记录，您应该在 `username` 上创建索引。请注意，虽然索引可以使 SELECT 查询快得多，但它会减慢 INSERT、UPDATE 和 DELETE 查询。

对于复杂的数据库查询，建议您创建数据库视图以节省查询解析和准备时间。

最后但同样重要的是，在 `SELECT` 查询中使用 `LIMIT`。这避免了从数据库中获取大量数据并耗尽分配给 PHP 的内存。

## 优化 Composer 自动加载器 <span id="optimizing-autoloader"></span>

因为 Composer 自动加载器用于包含大多数第三方类文件，所以您应该考虑通过执行以下命令来优化它：

```
composer dumpautoload -o
```

此外，您可以考虑使用[权威类映射](https://getcomposer.org/doc/articles/autoloader-optimization.md#optimization-level-2-a-authoritative-class-maps)和 [APCu 缓存](https://getcomposer.org/doc/articles/autoloader-optimization.md#optimization-level-2-b-apcu-cache)。请注意，这两种优化可能适合也可能不适合您的特定情况。

## 离线处理数据 <span id="processing-data-offline"></span>

当请求涉及一些资源密集型操作时，您应该考虑在离线模式下执行这些操作的方法，而不让用户等待它们完成。

有两种离线处理数据的方法：拉取和推送。

在拉取方法中，每当请求涉及一些复杂操作时，您创建一个任务并将其保存在持久存储中，例如数据库。然后使用单独的进程（例如 cron 作业）来拉取任务并处理它们。这种方法实现起来很简单，但它有一些缺点。例如，任务进程需要定期从任务存储中拉取。如果拉取频率太低，任务可能会被延迟处理，但如果频率太高，它会引入高开销。

在推送方法中，您将使用消息队列（例如 RabbitMQ、ActiveMQ、Amazon SQS 等）来管理任务。每当新任务放入队列时，它将启动或通知任务处理进程以触发任务处理。

## 使用预加载

从 PHP 7.4.0 开始，PHP 可以配置为在引擎启动时将脚本预加载到 opcache 中。您可以在[文档](https://www.php.net/manual/en/opcache.preloading.php)和相应的 [RFC](https://wiki.php.net/rfc/preload) 中阅读更多信息。

请注意，性能和内存之间的最佳权衡可能因应用而异。"预加载所有内容"可能是最简单的策略，但不一定是最好的策略。

例如，我们对简单的 [yiisoft/app](https://github.com/yiisoft/app) 应用模板进行了基准测试。在没有预加载和预加载整个 composer 类映射的情况下。

### 预加载基准测试

应用模板基准测试包括在引导脚本中配置类以注入依赖项。

对于两种变体，都使用了 [ApacheBench](https://httpd.apache.org/docs/2.4/programs/ab.html)，运行参数如下：

```shell
ab -n 1000 -c 10 -t 10
```

此外，调试模式已禁用。并且使用了 [Composer](https://getcomposer.org) 的优化自动加载器，并且没有使用开发依赖项：

```shell
composer install --optimize-autoloader --no-dev
```

启用预加载后，使用了整个 composer 类映射（825 个文件）：

```php
$files = require 'vendor/composer/autoload_classmap.php';

foreach (array_unique($files) as $file) {
    opcache_compile_file($file);
}
```

#### 测试结果

| 基准测试 | 预加载文件数 | Opcache 内存使用 | 每个请求内存使用 | 每个请求时间 | 每秒请求数 |
|--------------------|-----------------|---------------------|-------------------------|------------------|---------------------|
| 没有预加载 | 0               | 12.32 mb            | 1.71 mb                 | 27.63 ms         | 36.55 rq/s          |
| 有预加载    | 825             | 17.86 mb            | 1.82 mb                 | 26.21 ms         | 38.42 rq/s          |

如您所见，测试结果没有太大差异，因为这只是一个包含少数类的干净应用模板。有关预加载的更多讨论（包括基准测试）可以在 [composer 的 issue](https://github.com/composer/composer/issues/7777) 中找到。

## 性能分析 <span id="performance-profiling"></span>

您应该分析代码以找出性能瓶颈并采取相应的适当措施。以下分析工具可能有用：

<!-- - [Yii debug toolbar and debugger](https://github.com/yiisoft/yii2-debug/blob/master/docs/guide/README.md) -->

- [Blackfire](https://blackfire.io/)
- [XHProf](https://secure.php.net/manual/en/book.xhprof.php)
- [XDebug profiler](https://xdebug.org/docs/profiler)
