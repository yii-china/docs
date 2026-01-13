# 应用程序

Yii3 中 Web 应用程序及其运行器的主要目的是处理请求以获取响应。

通常，运行时包括：

1. 启动。获取配置，创建容器实例并进行额外的环境初始化，例如注册错误处理器，以便它可以处理发生的错误。触发 `ApplicationStartup` 事件。
2. 通过将请求对象传递给中间件调度器来处理请求，以执行[中间件堆栈](middleware.md)并获取响应对象。在通常的 PHP 应用程序中，这只执行一次。在[诸如 RoadRunner 之类的环境](../tutorial/using-with-event-loop.md)中，可以使用同一个应用程序实例多次执行。响应对象通过使用发射器转换为实际的 HTTP 响应。触发 `AfterEmit` 事件。
3. 关闭。触发 `ApplicationShutdown` 事件。
