# 脚本、样式和元标签

现代 Web 应用需要仔细管理 CSS 样式、JavaScript 代码和 HTML 元标签。Yii3 通过 `WebView` 类提供了一个全面的系统来注册和组织这些资源，该类是 `yiisoft/view` 包的一部分。

## 概述

`WebView` 类扩展了基本的 `View` 类，增加了 Web 特定的功能，允许您：

- 注册 CSS 文件和内联样式
- 注册 JavaScript 文件和内联脚本  
- 管理 HTML 元标签和 link 标签
- 控制资源渲染的位置
- 处理资源之间的依赖关系

## CSS 管理

### 注册 CSS 文件

您可以注册要包含在 HTML 页面中的 CSS 文件：

```php
<?php

declare(strict_types=1);

use Yiisoft\View\WebView;

/**
 * @var WebView $this
 */

// 注册 CSS 文件
$this->registerCssFile('/css/styles.css');

// 注册带属性的 CSS 文件
$this->registerCssFile('/css/print.css', WebView::POSITION_HEAD, [
    'media' => 'print',
]);

// 使用自定义键注册 CSS 文件以避免重复
$this->registerCssFile('/css/theme.css', WebView::POSITION_HEAD, [], 'theme-css');
```

### 注册内联 CSS

对于内联 CSS 样式，使用 `registerCss()` 方法：

```php
// 注册内联 CSS
$this->registerCss('
    .highlight {
        background-color: yellow;
        font-weight: bold;
    }
    .error {
        color: red;
        border: 1px solid red;
    }
', WebView::POSITION_HEAD);

// 带自定义属性
$this->registerCss('
    @media print {
        .no-print { display: none; }
    }
', WebView::POSITION_HEAD, ['id' => 'print-styles']);
```

### 从文件注册 CSS

您也可以从外部文件注册 CSS 内容：

```php
// 从文件读取 CSS 并注册为内联 CSS
$this->registerCssFromFile('/path/to/dynamic-styles.css', WebView::POSITION_HEAD, [
    'id' => 'dynamic-styles',
]);
```

### 使用样式标签

为了更好地控制，您可以直接使用 HTML 样式标签：

```php
use Yiisoft\Html\Html;

$styleTag = Html::style('
    .custom-button {
        background: linear-gradient(45deg, #blue, #purple);
        border: none;
        color: white;
        padding: 10px 20px;
    }
', ['id' => 'custom-button-styles']);

$this->registerStyleTag($styleTag, WebView::POSITION_HEAD);
```

## JavaScript 管理

### 注册 JavaScript 文件

使用 `registerJsFile()` 包含外部 JavaScript 文件：

```php
// 注册 JavaScript 文件
$this->registerJsFile('/js/main.js');

// 注册带属性的文件（异步加载）
$this->registerJsFile('/js/analytics.js', WebView::POSITION_END, [
    'async' => true,
]);

// 注册带 defer 属性的文件
$this->registerJsFile('/js/interactive.js', WebView::POSITION_END, [
    'defer' => true,
]);

// 从 CDN 注册
$this->registerJsFile('https://cdn.jsdelivr.net/npm/jquery@3.6.0/dist/jquery.min.js', 
    WebView::POSITION_END, [], 'jquery');
```

### 注册内联 JavaScript

使用 `registerJs()` 添加内联 JavaScript 代码：

```php
// 注册内联 JavaScript
$this->registerJs('
    document.addEventListener("DOMContentLoaded", function() {
        console.log("Page loaded!");
        initializeComponents();
    });
', WebView::POSITION_END);

// 注册应在 DOM 准备就绪时运行的 JavaScript
$this->registerJs('
    function showAlert(message) {
        alert(message);
    }
', WebView::POSITION_READY);
```

### JavaScript 变量

使用 `registerJsVar()` 将 PHP 数据传递给 JavaScript：

```php
// 注册 JavaScript 变量
$this->registerJsVar('apiUrl', 'https://api.example.com');
$this->registerJsVar('currentUser', [
    'id' => $user->getId(),
    'name' => $user->getName(),
    'isAdmin' => $user->isAdmin(),
]);
$this->registerJsVar('config', [
    'debug' => $this->isDebugMode(),
    'locale' => $this->getLocale(),
]);
```

这会生成如下 JavaScript 代码：

```html
<script>
var apiUrl = "https://api.example.com";
var currentUser = {"id":123,"name":"John Doe","isAdmin":false};
var config = {"debug":true,"locale":"en-US"};
</script>
```

### 使用脚本标签

为了更好地控制脚本标签：

```php
use Yiisoft\Html\Html;

$scriptTag = Html::script('
    window.myApp = {
        init: function() {
            console.log("App initialized");
        }
    };
', ['type' => 'module']);

$this->registerScriptTag($scriptTag, WebView::POSITION_END);
```

## 位置常量

资源可以放置在 HTML 文档的不同位置：

```php
use Yiisoft\View\WebView;

// 在 <head> 部分
WebView::POSITION_HEAD

// 在 <body> 开头
WebView::POSITION_BEGIN  

// 在 <body> 结尾（在 </body> 之前）
WebView::POSITION_END

// 当 DOM 准备就绪时（相当于 jQuery document.ready）
WebView::POSITION_READY

// 当页面完全加载时（相当于 window.onload）
WebView::POSITION_LOAD
```

显示每个位置渲染位置的示例布局：

```php
<?php

declare(strict_types=1);

/**
 * @var \Yiisoft\View\WebView $this
 */
?>
<?php $this->beginPage() ?>
<!DOCTYPE html>
<html>
<head>
    <!-- POSITION_HEAD 资源在此渲染 -->
    <?php $this->head() ?>
</head>
<body>
    <!-- POSITION_BEGIN 资源在此渲染 -->
    <?php $this->beginBody() ?>
    
    <main>
        <?= $content ?>
    </main>
    
    <!-- POSITION_END、POSITION_READY、POSITION_LOAD 资源在此渲染 -->
    <?php $this->endBody() ?>
</body>
</html>
<?php $this->endPage() ?>
```

## 元标签

### 基本元标签

为 SEO 和页面信息注册元标签：

```php
// 使用数组语法注册元标签
$this->registerMeta(['name' => 'description', 'content' => 'Page description']);
$this->registerMeta(['name' => 'keywords', 'content' => 'yii, php, framework']);
$this->registerMeta(['name' => 'author', 'content' => 'John Doe']);
$this->registerMeta(['name' => 'robots', 'content' => 'index, follow']);

// 响应式设计的视口
$this->registerMeta(['name' => 'viewport', 'content' => 'width=device-width, initial-scale=1']);

// 社交媒体的 Open Graph 标签
$this->registerMeta(['property' => 'og:title', 'content' => 'Page Title']);
$this->registerMeta(['property' => 'og:description', 'content' => 'Page description']);
$this->registerMeta(['property' => 'og:image', 'content' => 'https://example.com/image.jpg']);
```

### 使用元标签对象

为了更好地控制，使用 `Html::meta()` 辅助函数：

```php
use Yiisoft\Html\Html;

// 使用 Html 辅助函数创建元标签
$this->registerMetaTag(
    Html::meta()
        ->name('description')
        ->content('This is a comprehensive guide to Yii3 views')
);

$this->registerMetaTag(
    Html::meta()
        ->httpEquiv('refresh')
        ->content('300') // 每 5 分钟刷新一次
);
```

### 防止重复元标签

使用键来防止重复的元标签：

```php
// 第一次注册
$this->registerMeta([
    'name' => 'description',
    'content' => 'Original description'
], 'description');

// 这将覆盖前一个
$this->registerMeta([
    'name' => 'description', 
    'content' => 'Updated description'
], 'description');
```

## Link 标签

### 基本 Link 标签

注册各种类型的 link 标签：

```php
// 网站图标
$this->registerLink([
    'rel' => 'icon',
    'type' => 'image/png',
    'href' => '/favicon.png',
]);

// RSS 订阅
$this->registerLink([
    'rel' => 'alternate',
    'type' => 'application/rss+xml',
    'title' => 'RSS Feed',
    'href' => '/feed.rss',
]);

// 规范 URL
$this->registerLink([
    'rel' => 'canonical',
    'href' => 'https://example.com/canonical-url',
]);

// 预加载资源
$this->registerLink([
    'rel' => 'preload',
    'href' => '/fonts/main.woff2',
    'as' => 'font',
    'type' => 'font/woff2',
    'crossorigin' => 'anonymous',
]);
```

### 使用 Link 标签对象

```php
use Yiisoft\Html\Html;

// CSS 样式表
$this->registerLinkTag(
    Html::link('/css/main.css', [
        'rel' => 'stylesheet',
    ])
);

// DNS 预取
$this->registerLinkTag(
    Html::link('https://fonts.googleapis.com', [
        'rel' => 'dns-prefetch',
    ])
);
```

## 实际示例

### 完整的页面设置

以下是如何设置包含所有类型资源的完整页面：

```php
<?php

declare(strict_types=1);

use Yiisoft\Html\Html;
use Yiisoft\View\WebView;

/**
 * @var WebView $this
 * @var string $title
 * @var string $description
 * @var array $product
 */

// 设置页面标题
$this->setTitle($title);

// 元标签
$this->registerMeta(['name' => 'description', 'content' => $description]);
$this->registerMeta(['name' => 'keywords', 'content' => 'ecommerce, products, online shop']);

// Open Graph 标签
$this->registerMeta(['property' => 'og:title', 'content' => $title]);
$this->registerMeta(['property' => 'og:description', 'content' => $description]);
$this->registerMeta(['property' => 'og:image', 'content' => $product['image']]);

// CSS 文件
$this->registerCssFile('/css/product.css');
$this->registerCssFile('/css/responsive.css', WebView::POSITION_HEAD, [
    'media' => 'screen and (max-width: 768px)',
]);

// JavaScript 文件
$this->registerJsFile('/js/product-gallery.js', WebView::POSITION_END);
$this->registerJsFile('/js/shopping-cart.js', WebView::POSITION_END);

// JavaScript 变量
$this->registerJsVar('productData', $product);
$this->registerJsVar('cartApiUrl', '/api/cart');

// 内联 JavaScript
$this->registerJs('
    document.addEventListener("DOMContentLoaded", function() {
        initProductGallery();
        initShoppingCart();
    });
', WebView::POSITION_END);

// 页面特定样式
$this->registerCss('
    .product-special {
        background: linear-gradient(45deg, #ff6b6b, #4ecdc4);
        padding: 20px;
        border-radius: 10px;
    }
', WebView::POSITION_HEAD);
?>

<div class="product-page">
    <!-- 您的页面内容在此 -->
</div>
```

### 条件资源加载

根据条件加载资源：

```php
<?php

declare(strict_types=1);

/**
 * @var WebView $this
 * @var bool $isDarkMode
 * @var bool $isAdmin
 * @var string $userRole
 */

// 加载主题特定的 CSS
if ($isDarkMode) {
    $this->registerCssFile('/css/dark-theme.css');
} else {
    $this->registerCssFile('/css/light-theme.css');
}

// 管理员特定资源
if ($isAdmin) {
    $this->registerCssFile('/css/admin-toolbar.css');
    $this->registerJsFile('/js/admin-functions.js');
}

// 基于角色的 JavaScript 配置
$this->registerJsVar('userPermissions', [
    'canEdit' => in_array($userRole, ['admin', 'editor']),
    'canDelete' => $userRole === 'admin',
    'canPublish' => in_array($userRole, ['admin', 'publisher']),
]);
?>
```

## 最佳实践

1. **使用适当的位置**：将 CSS 放在 `POSITION_HEAD`，JavaScript 放在 `POSITION_END`
2. **最小化内联资源**：优先使用外部文件以获得更好的缓存
3. **使用键避免重复**：使用有意义的键防止重复资源
4. **优化加载**：对非关键 JavaScript 使用 `async` 和 `defer` 属性
5. **分组相关资源**：将相关的 CSS 和 JS 文件放在一起
6. **明智使用 CDN**：平衡性能与可靠性
7. **验证元标签**：确保设置了适当的 SEO 元标签
8. **考虑安全性**：小心使用内联脚本和 CSP 策略

## 使用资源包

对于更复杂的资源管理，考虑使用资源包：

```php
// 注册资源包（在资源指南中详细介绍）
$assetBundle = $this->assetManager->register(MainAsset::class);

// 从包中添加所有 CSS 文件
$this->addCssFiles($this->assetManager->getCssFiles());

// 从包中添加所有 JavaScript 文件  
$this->addJsFiles($this->assetManager->getJsFiles());
```
