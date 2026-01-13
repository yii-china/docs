# 资源管理

资源管理对于现代 Web 应用至关重要。资源包括 CSS 样式表、JavaScript 文件、图片、字体和其他静态资源。Yii3 通过 `yiisoft/assets` 包提供了一个全面的资源管理系统，用于处理这些资源的依赖关系、优化和部署。

## 安装

资源管理功能由 `yiisoft/assets` 包提供：

```bash
composer require yiisoft/assets
```

此包默认包含在 `yiisoft/app` 应用模板中。

## 基本概念

### 资源包

资源包是相关资源文件（CSS、JavaScript、图片）的集合，它们在逻辑上组合在一起。资源包可以依赖于其他包，从而实现适当的依赖管理。

### 资源管理器

资源管理器负责：
- 解析资源包依赖关系
- 将资源从受保护的目录发布到 Web 可访问的位置
- 组合和压缩资源（配置后）
- 为资源生成适当的 URL

## 创建资源包

### 基本资源包

这是一个简单的资源包定义：

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class MainAsset extends AssetBundle
{
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $css = [
        'css/main.css',
        'css/responsive.css',
    ];
    
    public array $js = [
        'js/main.js',
        'js/utils.js',
    ];
    
    public array $depends = [
        BootstrapAsset::class,
        JqueryAsset::class,
    ];
}
```

### 资源包属性

**路径配置：**
- `$basePath` - 资源文件所在的物理路径
- `$baseUrl` - 资源的 Web 可访问 URL 路径
- `$sourcePath` - 需要发布的资源的源目录

**资源文件：**
- `$css` - CSS 文件数组
- `$js` - JavaScript 文件数组  

**依赖关系：**
- `$depends` - 此包依赖的其他资源包数组

**选项：**
- `$jsOptions` - JavaScript 标签的 HTML 属性
- `$cssOptions` - CSS link 标签的 HTML 属性

### 高级资源包

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class AdminAsset extends AssetBundle
{
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    // 带媒体查询的 CSS 文件
    public array $css = [
        'css/admin/main.css',
        ['css/admin/print.css', 'media' => 'print'],
        ['css/admin/mobile.css', 'media' => 'screen and (max-width: 768px)'],
    ];
    
    // 带属性的 JavaScript 文件
    public array $js = [
        'js/admin/core.js',
        ['js/admin/charts.js', 'defer' => true],
        ['js/admin/analytics.js', 'async' => true],
    ];
    
    // 此包中所有 JS 文件的全局选项
    public array $jsOptions = [
        'data-admin' => 'true',
    ];
    
    // 此包中所有 CSS 文件的全局选项
    public array $cssOptions = [
        'data-bundle' => 'admin',
    ];
    
    public array $depends = [
        MainAsset::class,
        ChartJsAsset::class,
    ];
}
```

## 使用资源包

### 在控制器中

在控制器或视图中注册资源包：

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Asset\MainAsset;
use Psr\Http\Message\ResponseInterface;
use Yiisoft\Assets\AssetManager;
use Yiisoft\Yii\View\Renderer\ViewRenderer;

final class SiteController
{
    public function __construct(
        private ViewRenderer $viewRenderer,
        private AssetManager $assetManager,
    ) {}

    public function index(): ResponseInterface
    {
        // 注册资源包
        $this->assetManager->register(MainAsset::class);
        
        return $this->viewRenderer->render('index', [
            'title' => 'Home Page',
        ]);
    }
    
    public function admin(): ResponseInterface
    {
        // 注册多个资源包
        $this->assetManager->register([
            MainAsset::class,
            AdminAsset::class,
        ]);
        
        return $this->viewRenderer->render('admin/dashboard');
    }
}
```

### 在视图中

您也可以直接在视图中注册资源：

```php
<?php

declare(strict_types=1);

use App\Asset\ProductAsset;
use Yiisoft\Assets\AssetManager;

/**
 * @var \Yiisoft\View\WebView $this
 * @var AssetManager $assetManager
 * @var array $product
 */

// 注册产品特定的资源
$assetManager->register(ProductAsset::class);
?>

<div class="product-page">
    <h1><?= Html::encode($product['name']) ?></h1>
    <!-- 产品内容 -->
</div>
```

### 与 WebView 集成

推荐的方法是与 WebView 集成以实现自动资源渲染：

```php
<?php

declare(strict_types=1);

use App\Asset\MainAsset;

/**
 * @var \Yiisoft\View\WebView $this
 * @var \Yiisoft\Assets\AssetManager $assetManager
 */

// 注册资源
$assetManager->register(MainAsset::class);

// 将所有已注册的资源添加到视图
$this->addCssFiles($assetManager->getCssFiles());
$this->addCssStrings($assetManager->getCssStrings());
$this->addJsFiles($assetManager->getJsFiles());
$this->addJsStrings($assetManager->getJsStrings());
$this->addJsVars($assetManager->getJsVars());
?>

<div class="page-content">
    <!-- 您的页面内容 -->
</div>
```

## 资源发布

### 源路径发布

当资源位于非 Web 可访问的目录（如 vendor 包）时，需要发布它们：

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class VendorAsset extends AssetBundle
{
    // 源目录（不可 Web 访问）
    public string $sourcePath = '@vendor/company/package/assets';
    
    // 将发布到 Web 可访问目录
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $css = [
        'styles.css',
    ];
    
    public array $js = [
        'script.js',
    ];
}
```

### 自定义发布

您也可以手动发布目录：

```php
/**
 * @var \Yiisoft\Assets\AssetManager $assetManager
 */

// 发布目录
$publishedPath = $assetManager->publish('@vendor/company/package/assets');

// 获取已发布的 URL
$publishedUrl = $assetManager->getPublishedUrl('@vendor/company/package/assets');
```

## 第三方库资源

### jQuery 资源包

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class JqueryAsset extends AssetBundle
{
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $js = [
        'js/jquery-3.6.0.min.js',
    ];
    
    // 或使用 CDN
    public array $jsOptions = [
        'integrity' => 'sha256-...',
        'crossorigin' => 'anonymous',
    ];
}
```

### Bootstrap 资源包

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class BootstrapAsset extends AssetBundle
{
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $css = [
        'css/bootstrap.min.css',
    ];
    
    public array $js = [
        'js/bootstrap.bundle.min.js',
    ];
    
    public array $depends = [
        JqueryAsset::class, // Bootstrap 需要 jQuery
    ];
}
```

### CDN 资源

对于 CDN 托管的资源，您可以指定完整的 URL：

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class CdnAsset extends AssetBundle
{
    public array $css = [
        'https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css',
    ];
    
    public array $js = [
        'https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js',
    ];
    
    public array $jsOptions = [
        'integrity' => 'sha384-ka7Sk0Gln4gmtz2MlQnikT1wXgYsOg+OMhuP+IlRH9sENBO0LRn5q+8nbTov4+1p',
        'crossorigin' => 'anonymous',
    ];
}
```

## 资源配置

### 应用配置

在应用配置中配置资源管理：

**config/web/params.php**
```php
return [
    'yiisoft/assets' => [
        // 已发布资源的基本路径
        'basePath' => '@webroot/assets',
        
        // 资源的基本 URL
        'baseUrl' => '@web/assets',
        
        // 资源转换器配置
        'converter' => [
            'commands' => [
                'scss' => ['sass', '{from}', '{to}', '--style=compressed'],
                'ts' => ['tsc', '{from}', '--outFile', '{to}'],
            ],
        ],
    ],
];
```

### 环境特定资源

为不同环境配置不同的资源：

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class MainAsset extends AssetBundle
{
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $css = [];
    public array $js = [];
    
    public function __construct()
    {
        if (YII_ENV_DEV) {
            // 开发资源（未压缩）
            $this->css = ['css/main.css', 'css/debug.css'];
            $this->js = ['js/main.js', 'js/debug.js'];
        } else {
            // 生产资源（已压缩）
            $this->css = ['css/main.min.css'];
            $this->js = ['js/main.min.js'];
        }
    }
}
```

## 资源优化

### 资源组合

将多个 CSS 或 JavaScript 文件组合成单个文件：

```php
/**
 * @var \Yiisoft\Assets\AssetManager $assetManager
 */

// 启用资源组合
$assetManager->setCombine(true);

// 设置组合选项
$assetManager->setCombineOptions([
    'css' => true,  // 组合 CSS 文件
    'js' => true,   // 组合 JavaScript 文件
]);
```

### 资源压缩

为生产环境配置资源压缩：

```php
// 在资源管理器配置中
'converter' => [
    'commands' => [
        'css' => ['cleancss', '{from}', '-o', '{to}'],
        'js' => ['uglifyjs', '{from}', '-o', '{to}', '--compress', '--mangle'],
    ],
],
```

## 使用资源转换器

### SCSS/SASS 编译

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class ScssAsset extends AssetBundle
{
    public string $sourcePath = '@app/assets/scss';
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $css = [
        'main.scss', // 将被转换为 main.css
        'components.scss',
    ];
}
```

### TypeScript 编译

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

final class TypeScriptAsset extends AssetBundle
{
    public string $sourcePath = '@app/assets/ts';
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $js = [
        'main.ts', // 将被转换为 main.js
        'utils.ts',
    ];
}
```

## 实际示例

### 完整的应用资源结构

```
assets/
├── css/
│   ├── main.css           # 主应用样式
│   ├── admin.css          # 管理员特定样式
│   └── mobile.css         # 移动端特定样式
├── js/
│   ├── main.js            # 主应用 JavaScript
│   ├── admin.js           # 管理员功能
│   └── vendor/            # 第三方库
│       ├── jquery.min.js
│       └── bootstrap.min.js
├── images/
│   ├── logo.png
│   └── icons/
└── fonts/
    ├── main.woff2
    └── main.woff
```

### 完整的资源包设置

```php
<?php

declare(strict_types=1);

namespace App\Asset;

use Yiisoft\Assets\AssetBundle;

// 基础资源包
final class AppAsset extends AssetBundle
{
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $css = [
        'css/main.css',
    ];
    
    public array $js = [
        'js/main.js',
    ];
    
    public array $depends = [
        JqueryAsset::class,
    ];
}

// 管理员特定包
final class AdminAsset extends AssetBundle
{
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $css = [
        'css/admin.css',
    ];
    
    public array $js = [
        'js/admin.js',
    ];
    
    public array $depends = [
        AppAsset::class,
        BootstrapAsset::class,
    ];
}

// 移动端特定包  
final class MobileAsset extends AssetBundle
{
    public string $basePath = '@assets';
    public string $baseUrl = '@assetsUrl';
    
    public array $css = [
        ['css/mobile.css', 'media' => 'screen and (max-width: 768px)'],
    ];
    
    public array $depends = [
        AppAsset::class,
    ];
}
```

## 最佳实践

1. **按功能组织**：将相关资源分组到逻辑包中
2. **管理依赖关系**：正确声明包之间的依赖关系
3. **使用有意义的名称**：清晰地命名您的资源包
4. **环境优化**：在生产环境中使用压缩资源
5. **考虑 CDN**：适当时为流行库使用 CDN
6. **版本化资源**：在文件名中包含版本号以实现缓存清除
7. **最小化 HTTP 请求**：尽可能组合相关资源
8. **优化文件大小**：为生产环境压缩和精简资源

## 故障排除

### 常见问题

**找不到资源：**
- 检查资源文件是否存在于指定路径
- 验证 `$basePath` 和 `$baseUrl` 配置是否正确
- 如果使用 `$sourcePath`，确保资源已发布

**依赖项未加载：**
- 验证依赖包是否正确注册
- 检查是否存在循环依赖
- 确保依赖包配置正确

**资源未出现在 HTML 中：**
- 确保在布局中调用了 `$this->addCssFiles()` 和 `$this->addJsFiles()`
- 验证在布局中调用了 `$this->head()`、`$this->beginBody()` 和 `$this->endBody()`

**权限问题：**
- 检查 Web 服务器是否对资源目录有写权限
- 验证已发布的资源目录是否可 Web 访问
