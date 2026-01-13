# 视图

视图负责向最终用户呈现数据。你给它一个带有占位符和呈现逻辑的模板以及一些数据。视图将数据传递给模板并执行模板逻辑。最终结果可以传递给最终用户，无论是浏览器、要下载的文件、要发送的电子邮件还是其他内容。

```mermaid
flowchart LR
  A[Data] --> V[View]
  B[Template] --> V
  V --> C[HTML]
  C --> D[Browser]
```

在 Yii3 中，视图通常是包含呈现逻辑和 HTML 标记的 PHP 文件。视图系统提供了一种灵活的方式来组织你的呈现层，并支持布局和部分视图等功能。除了使用纯 PHP 模板，你还可以利用[模板引擎（如 Twig）](template-engines.md)。

## 安装

对于基本的视图功能，你需要 [`yiisoft/view`](https://github.com/yiisoft/view) 包：

```sh
composer require yiisoft/view
```

对于 Web 应用程序，你还应该安装 [`yiisoft/yii-view-renderer`](https://github.com/yiisoft/yii-view-renderer) 包，它提供了 [PSR-7](https://www.php-fig.org/psr/psr-7/) 兼容性和 Web 特定功能：

```sh
composer require yiisoft/yii-view-renderer
```

这些包默认包含在 [`yiisoft/app`](https://github.com/yiisoft/app) 应用程序模板中。

## 基本概念

视图模板文件包含呈现逻辑。
在 [`yiisoft/app`](https://github.com/yiisoft/app) 模板中，视图文件通常与其控制器存储在一起（例如，[`src/Web/Echo/Action.php`](../../guide/structure/action.md)）。以下是一个简单的视图文件示例，[`src/Web/Echo/template.php`](../../guide/structure/action.md)：

```php
<?php
use Yiisoft\Html\Html;
/* @var string $message */
?>

<p>The message is: <?= Html::encode($message) ?></p>
```

这里 [`$message`](../../guide/structure/action.md) 是在使用 [`ViewRenderer`](https://github.com/yiisoft/yii-view-renderer) 渲染模板时传递的视图数据。例如，[`src/Web/Echo/Action.php`](../../guide/structure/action.md)：

```php
<?php

declare(strict_types=1);

namespace App\Web\Echo;

use Psr\Http\Message\ResponseInterface;
use Yiisoft\Router\HydratorAttribute\RouteArgument;
use Yiisoft\Yii\View\Renderer\ViewRenderer;

final readonly class Action
{
    public function __construct(
        private ViewRenderer $viewRenderer,
    ) {}

    public function __invoke(
        #[RouteArgument('message')]
        string $message = 'Hello!'
    ): ResponseInterface
    {
        return $this->viewRenderer->render(__DIR__ . '/template', [
            'message' => $message,
        ]);
    }
}
```

[`render()`](https://github.com/yiisoft/yii-view-renderer) 方法的第一个参数是模板文件的路径。在 [`yiisoft/app`](https://github.com/yiisoft/app) 中，模板文件通常与其操作存储在一起。结果已准备好渲染到浏览器，因此我们立即返回它。

## 使用布局

大多数 Web 应用程序对所有页面使用通用布局。在 [`yiisoft/app`](https://github.com/yiisoft/app) 模板中，布局存储在 [`src/Web/Shared/Layout/Main/`](../../guide/structure/application.md) 目录中。你可以在 [`config/common/params.php`](../../guide/concept/configuration.md) 中设置默认布局：

```php
return [
    'yiisoft/yii-view-renderer' => [
        'viewPath' => null,
        'layout' => '@src/Web/Shared/Layout/Main/layout.php',
    ],
];
```

典型的布局文件（如 [`src/Web/Shared/Layout/Main/layout.php`](../../guide/structure/application.md)）如下所示：

```php
<?php

declare(strict_types=1);

use App\Web\Shared\Layout\Main\MainAsset;
use Yiisoft\Html\Html;

/**
 * @var \App\Shared\ApplicationParams $applicationParams
 * @var Yiisoft\Aliases\Aliases $aliases
 * @var Yiisoft\Assets\AssetManager $assetManager
 * @var string $content
 * @var string|null $csrf
 * @var Yiisoft\View\WebView $this
 * @var Yiisoft\Router\CurrentRoute $currentRoute
 * @var Yiisoft\Router\UrlGeneratorInterface $urlGenerator
 */

$assetManager->register(MainAsset::class);

$this->addCssFiles($assetManager->getCssFiles());
$this->addCssStrings($assetManager->getCssStrings());
$this->addJsFiles($assetManager->getJsFiles());
$this->addJsStrings($assetManager->getJsStrings());
$this->addJsVars($assetManager->getJsVars());

$this->beginPage()
?>
<!DOCTYPE html>
<html lang="<?= Html::encode($applicationParams->locale) ?>">
<head>
    <meta charset="<?= Html::encode($applicationParams->charset) ?>">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="icon" href="<?= $aliases->get('@baseUrl/favicon.svg') ?>" type="image/svg+xml">
    <title><?= Html::encode($this->getTitle()) ?></title>
    <?php $this->head() ?>
</head>
<body>
<?php $this->beginBody() ?>

<div class="header">
    <a href="/">
        <?= Html::encode($applicationParams->name) ?>
    </a>
</div>

<div class="content">    
    <?= $content ?>
</div>

<div class="footer">
    © <?= date('Y') ?>  <?= Html::encode($applicationParams->name) ?>    
</div>

<?php $this->endBody() ?>
</body>
</html>
<?php $this->endPage() ?>
```

在上面的模板中，[`$applicationParams`](../../guide/concept/configuration.md) 是来自 [`config/common/application.php`](../../guide/concept/configuration.md) 的参数数组。

[`$aliases`](../concept/aliases.md) 指的是[别名组件](../concept/aliases.md)，用于获取服务器上应用程序的基本 URL。

[`$this`](https://github.com/yiisoft/view) 是视图的实例，我们使用它来获取页面标题和输出资源。包括标准和自定义资源。

通过 [`$assetManager->register(MainAsset::class);`](asset.md) 我们注册一个定义要包含到页面的 [`css`](asset.md) 的资源。它在首次使用时会自动复制到 [`public/assets`](asset.md)。对于单个 CSS 文件来说不是很有用，但随着资源数量的增长会变得很方便。

> [!IMPORTANT]
> 纯 PHP 模板中的输出不会被编码，你不应该忘记使用 [`Html::encode()`](https://github.com/yiisoft/html) 来防止 XSS 安全漏洞。

有关布局中可用内容的更多信息，可以在 [yiisoft/view](https://github.com/yiisoft/view) 文档中阅读。

### 不使用布局渲染

有时你需要在不使用布局的情况下渲染视图（例如，对于 AJAX 响应）：

```php
public function ajaxContent(): ResponseInterface
{
    return $this->viewRenderer
        ->withLayout(null)
        ->render('ajax-content', ['data' => $data]);
}

// 或使用 renderPartial 方法
public function ajaxContent(): ResponseInterface
{
    return $this->viewRenderer->renderPartial('ajax-content', ['data' => $data]);
}
```

## 嵌套视图和部分视图

### 渲染子视图

你可以使用 [`$this->render()`](https://github.com/yiisoft/view) 方法在视图中渲染其他视图：

```php
<?php

declare(strict_types=1);

/**
 * @var \Yiisoft\View\WebView $this
 * @var \App\Entity\Post[] $posts
 */
?>
<div class="posts">
    <?php foreach ($posts as $post): ?>
        <?= $this->render('_post_item', ['post' => $post]) ?>
    <?php endforeach; ?>
</div>
```

**src/Web/Post/_item.php**
```php
<?php

declare(strict_types=1);

use Yiisoft\Html\Html;

/**
 * @var \Yiisoft\View\WebView $this
 * @var \App\Entity\Post $post
 */
?>
<article class="post">
    <h2><?= Html::encode($post->getTitle()) ?></h2>
    <p><?= Html::encode($post->getExcerpt()) ?></p>
    <time><?= $post->getCreatedAt()->format('Y-m-d') ?></time>
</article>
```

### 使用块

块允许你在一个视图中定义内容并在另一个视图中显示它，通常在布局中：

```php
// 在视图文件中
<?php $this->setBlock('sidebar', $this->render('_sidebar', ['items' => $sidebarItems])) ?>

// 在布局文件中
<?php if ($this->hasBlock('sidebar')): ?>
    <aside><?= $this->getBlock('sidebar') ?></aside>
<?php endif ?>
```

## 渲染为字符串

如果你需要将渲染的内容作为字符串而不是 PSR-7 响应：

```php
public function getEmailContent(): string
{
    return $this->viewRenderer->renderAsString('email/welcome', [
        'user' => $user,
    ]);
}
```

## 视图事件

视图系统在渲染过程中触发你可以监听的事件：

```php
use Yiisoft\View\Event\WebView\BeforeRender;
use Yiisoft\View\Event\WebView\AfterRender;

// 事件监听器示例
final class ViewEventListener
{
    public function onBeforeRender(BeforeRender $event): void
    {
        // 根据视图名称添加全局 CSS 类
        if (str_contains($event->getFile(), 'admin/')) {
            $event->getView()->registerCssClass('admin-view');
        }
    }

    public function onAfterRender(AfterRender $event): void
    {
        // 记录渲染时间
        $this->logger->info('View rendered', [
            'view' => $event->getFile(),
            'time' => $event->getRenderTime(),
        ]);
    }
}
```
