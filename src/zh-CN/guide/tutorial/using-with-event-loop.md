# 在事件循环中使用 Yii

正常的 PHP Web 请求执行周期包括设置环境、获取请求、处理它以形成响应并发送结果。响应发送后，执行终止并且其上下文丢失。因此，对于后续请求，整个序列会重复。这种方法在开发便利性方面有很大优势，因为开发人员不必太关心内存泄漏或正确清理上下文。另一方面，为每个请求初始化所有内容需要时间，总体上消耗高达 50% 的处理资源。

有一种运行应用的替代方法。事件循环。其思想是一次性初始化所有可能的内容，然后使用它处理多个请求。这种方法通常称为事件循环。

有多种工具可用于实现它。值得注意的是，[FrankenPHP](https://frankenphp.dev/)、[RoadRunner](https://roadrunner.dev/) 和 [Swoole](https://www.swoole.co.uk/)。

## 事件循环的影响

事件循环工作进程基本上如下所示：

```php
initializeContext();
while ($request = getRequest()) {
   $response = process($request);
   emit($response);
}
```

通常，有多个工作进程同时处理请求，就像传统的 php-fpm 一样。

这意味着在开发应用时需要考虑更多因素。

### 处理是阻塞的

工作进程逐个处理请求，当前处理会阻塞下一个请求的处理。这意味着长时间运行的进程，与一般 PHP 应用一样，应该通过使用队列放入后台。

### 服务和状态

由于事件循环中的上下文在单个工作进程处理的所有请求-响应之间共享，因此前一个请求对服务状态所做的所有更改都可能影响当前请求。此外，如果一个用户的数据对另一个用户可用，这可能是一个安全问题。

有两种方法可以处理它。首先，您可以通过使服务无状态来避免拥有状态。PHP 的 `readonly` 关键字可能对此很有用。其次，您可以在请求处理结束时重置服务的状态。在这种情况下，状态重置器将帮助您：

```php
initializeContext();
$resetter = $container->get(\Yiisoft\Di\StateResetter::class);
while ($request = getRequest()) {
   $response = process($request);
   emit($response);
   $resetter->reset(); // 我们应该在每个请求上重置此类服务的状态。
}
```

## 集成

- [RoadRunner](using-yii-with-roadrunner.md)
- [Swoole](using-yii-with-swoole.md)
