# 服务组件

应用程序可能会变得复杂，因此将业务逻辑或基础设施的重点部分提取到服务组件中是有意义的。它们通常被注入到其他组件或操作处理程序中。这通常通过自动装配完成：

```php
public function actionIndex(CurrentRoute $route, MyService $myService): ResponseInterface
{
    $id = $route->getArgument('id');
    
    // ...
    $extraData = $myService->getExtraData($id);
    
    // ...
}
```

Yii3 在技术上对如何构建服务没有任何限制。通常，不需要从基类扩展或实现某个接口：

```php
final readonly class MyService
{
    public function __construct(
        private ExtraDataStorage $extraDataStorage
    )
    {
    }

    public function getExtraData(string $id): array
    {
        return $this->extraDataStorage->get($id);
    }
}
```

服务要么执行任务，要么返回数据。它们被创建一次，放入 DI 容器中，然后可以多次使用。因此，保持服务无状态是一个好主意，即服务本身及其任何依赖项都不应该保持状态。你可以通过在类级别使用 `readonly` PHP 关键字来确保这一点。

## 服务依赖项和配置

服务应该始终通过 `__construct()` 定义它们对其他服务的所有依赖项。这既允许你在创建服务后立即使用它，又可以作为服务做太多事情的指标（如果有太多依赖项）。

- 创建服务后，不应在运行时重新配置它。
- DI 容器实例通常**不应该**作为依赖项注入。优先使用具体的接口。
- 在复杂或"繁重"的初始化情况下，尝试将其推迟到调用服务方法时。

配置值也是如此。它们应该作为构造函数参数提供。相关的值可以组合到值对象中。例如，数据库连接通常需要 DSN 字符串、用户名和 password。这三个可以组合到 Dsn 类中：

```php
final readonly class Dsn
{
    public function __construct(
        public string $dsn,
        public string $username,
        public string $password
    )
    {
        if (!$this->isValidDsn($dsn)) {
            throw new \InvalidArgumentException('DSN provided is not valid.');
        }
    }
    
    private function isValidDsn(string $dsn): bool
    {
        // 检查 DSN 有效性
    }
}
```

## 服务方法

服务方法通常做某事。它可以是一个简单的重复执行的事情，但通常它取决于上下文。例如：

```php
final readonly class PostPersister
{
    public function __construct(
        private Storage $db
    )
    {
    }
    
    public function persist(Post $post)
    {
        $this->db->insertOrUpdate('post', $post);
    }
}
```

有一个服务将帖子保存到永久存储（如数据库）中。允许与具体存储通信的对象始终相同，因此使用构造函数注入，而保存的帖子可能会有所不同，因此作为方法参数传递。

## 是否所有东西都是服务？

通常，选择另一种类类型来放置你的代码是有意义的。检查：

- Repository（仓储）
- Widget（小部件）
- [Middleware（中间件）](middleware.md)
- Entity（实体）
