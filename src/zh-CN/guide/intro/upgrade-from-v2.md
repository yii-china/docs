# 从版本 2.0 升级

> 如果你没有使用过 Yii 2.0，可以跳过本节，直接进入"[入门](../start/prerequisites.md)"部分。

虽然共享一些共同的理念和价值观，但 Yii3 在概念上与 Yii 2.0 不同。没有简单的升级路径，因此首先[查看 Yii 2.0 的维护策略和生命周期结束日期](https://www.yiiframework.com/release-cycle)，并考虑在 Yii3 上启动新项目，同时将现有项目保留在 Yii 2.0 上。

## PHP 要求

Yii3 需要 PHP 8.2 或更高版本。因此，使用了 Yii 2.0 中未使用的语言特性：

- [类型声明](https://www.php.net/manual/en/functions.arguments.php#functions.arguments.type-declaration)
- [返回类型声明](https://www.php.net/manual/en/functions.returning-values.php#functions.returning-values.type-declaration)
- [类常量可见性](https://www.php.net/manual/en/language.oop5.constants.php)
- [命名参数](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments)
- [匿名类](https://www.php.net/manual/en/language.oop5.anonymous.php)
- [::class](https://www.php.net/manual/en/language.oop5.basic.php#language.oop5.basic.class.class)
- [生成器](https://www.php.net/manual/en/language.generators.php)
- [可变参数函数](https://www.php.net/manual/en/functions.arguments.php#functions.variable-arg-list)
- [只读属性](https://www.php.net/manual/en/language.oop5.properties.php#language.oop5.properties.readonly-properties)
- [只读类](https://www.php.net/manual/en/language.oop5.basic.php#language.oop5.basic.class.readonly)
- [构造函数属性提升](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)
- [属性](https://www.php.net/manual/en/language.attributes.php)

## 初步重构

在将 Yii 2.0 项目移植到 Yii3 之前，重构它是一个好主意。这既可以使移植更容易，也可以在项目尚未迁移到 Yii3 时使其受益。

### 使用 DI 而不是服务定位器

由于 Yii3 强制你注入依赖项，因此准备并从使用服务定位器（`Yii::$app->`）切换到 [DI 容器](https://www.yiiframework.com/doc/guide/2.0/en/concept-di-container)是一个好主意。

如果由于某种原因使用 DI 容器有问题，请考虑将所有对 `Yii::$app->` 的调用移至控制器操作和小部件，并从控制器手动传递依赖项到需要它们的地方。

有关该思想的解释，请参阅[依赖注入和容器](../concept/di-container.md)。

### 引入用于获取数据的仓库

由于 Active Record 不是 Yii3 中使用数据库的唯一方式，请考虑引入仓库来隐藏获取数据的细节并将它们收集在一个地方。你可以稍后重做它：

```php
final readonly class PostRepository
{
    public function getArchive()
    {
        // ...
    }
    
    public function getTop10ForFrontPage()
    {
        // ...
    }

}
```

### 将领域层与基础设施分离

如果你有一个丰富复杂的领域，最好将其与框架提供的基础设施分离，即所有业务逻辑都应该放到与框架无关的类中。

### 将更多内容移入组件

Yii3 服务在概念上类似于 Yii 2.0 组件，因此将应用程序的可重用部分移入组件是一个好主意。

## 需要学习的内容

### Docker

默认应用程序模板使用 [Docker](https://www.docker.com/get-started/) 来运行应用程序。
学习如何使用它并将其用于你自己的项目是一个好主意，因为它提供了很多好处：

1. 与生产环境完全相同的环境。
2. 除了 Docker 本身之外，无需安装任何东西。
3. 环境是按应用程序而不是按服务器。

### 环境变量

Yii3 应用程序模板使用[环境变量](https://en.wikipedia.org/wiki/Environment_variable)来配置应用程序的部分内容。这个概念[对于 Docker 化的应用程序非常方便](https://12factor.net/)，但对于 Yii 1.1 和 Yii 2.0 的用户来说可能很陌生。

### 处理程序

与 Yii 2.0 不同，Yii3 不使用控制器。相反，它使用[处理程序](../structure/handler.md)，它们类似于控制器但又不同。

### 应用程序结构

建议的 Yii3 应用程序结构与 Yii 2.0 不同。
它在[应用程序结构](../structure/overview.md)中有描述。

尽管如此，Yii3 是灵活的，因此仍然可以在 Yii3 中使用类似于 Yii 2.0 的结构。
