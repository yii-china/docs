# 包

可重用的代码可以作为 [Composer 包](https://getcomposer.org/doc/05-repositories.md#package)发布。它可以是基础设施库、代表应用程序上下文之一的模块，或者基本上任何可重用的代码。

## 使用包 <span id="using-packages"></span>

默认情况下，Composer 安装在 [Packagist](https://packagist.org/) 上注册的包——这是最大的开源 PHP 包仓库。你可以在 Packagist 上查找包。你也可以[创建自己的仓库](https://getcomposer.org/doc/05-repositories.md#repository)并配置 Composer 使用它。如果你正在开发只想在项目内部共享的私有包，这会很有用。

Composer 安装的包存储在项目的 `vendor` 目录中。因为 Composer 是一个依赖管理器，当它安装一个包时，它也会安装该包的所有依赖包。

> [!WARNING]
> 应用程序的 `vendor` 目录永远不应该被修改。

可以使用以下命令安装包：

```
composer install vendor-name/package-name
```

完成后，Composer 会修改 `composer.json` 和 `composer.lock`。前者定义要安装的包及其版本约束，后者存储实际安装的确切版本的快照。

包中的类将通过[自动加载](../concept/autoloading.md)立即可用。

## 创建包 <span id="creating-packages"></span>

当你觉得需要与其他人分享你的优秀代码时，可以考虑创建一个包。包可以包含你喜欢的任何代码，例如辅助类、小部件、服务、中间件、整个模块等。

以下是你可以遵循的基本步骤。

1. 为你的包创建一个项目并将其托管在 VCS 仓库上，例如 [GitHub.com](https://github.com)。包的开发和维护工作应该在这个仓库上完成。
2. 在项目的根目录下，按照 Composer 的要求创建一个名为 `composer.json` 的文件。有关更多详细信息，请参阅下一小节。
3. 在 Composer 仓库（例如 [Packagist](https://packagist.org/)）中注册你的包，以便其他用户可以使用 Composer 找到并安装你的包。

### `composer.json` <span id="composer-json"></span>

每个 Composer 包必须在其根目录中有一个 `composer.json` 文件。该文件包含有关包的元数据。你可以在 [Composer 手册](https://getcomposer.org/doc/01-basic-usage.md#composer-json-project-setup)中找到有关此文件的完整规范。以下示例显示了 `yiisoft/yii-widgets` 包的 `composer.json` 文件：

```json
{
    "name": "yiisoft/yii-widgets",
    "type": "library",
    "description": "Yii widgets collection",
    "keywords": [
        "yii",
        "widgets"
    ],
    "homepage": "https://www.yiiframework.com/",
    "license": "BSD-3-Clause",
    "support": {
        "issues": "https://github.com/yiisoft/yii-widgets/issues?state=open",
        "forum": "https://www.yiiframework.com/forum/",
        "wiki": "https://www.yiiframework.com/wiki/",
        "irc": "ircs://irc.libera.chat:6697/yii",
        "chat": "https://t.me/yii3en",
        "source": "https://github.com/yiisoft/yii-widgets"
    },
    "funding": [
        {
            "type": "opencollective",
            "url": "https://opencollective.com/yiisoft"
        },
        {
            "type": "github",
            "url": "https://github.com/sponsors/yiisoft"
        }
    ],
    "require": {
        "php": "^7.4|^8.0",
        "yiisoft/aliases": "^1.1|^2.0",
        "yiisoft/cache": "^1.0",
        "yiisoft/html": "^2.0",
        "yiisoft/view": "^4.0",
        "yiisoft/widget": "^1.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^9.5",
        "roave/infection-static-analysis-plugin": "^1.16",
        "spatie/phpunit-watcher": "^1.23",
        "vimeo/psalm": "^4.18",
        "yiisoft/psr-dummy-provider": "^1.0",
        "yiisoft/test-support": "^1.3"
    },
    "autoload": {
        "psr-4": {
            "Yiisoft\\Yii\\Widgets\\": "src"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Yiisoft\\Yii\\Widgets\\Tests\\": "tests"
        }
    },
    "extra": {
        "branch-alias": {
            "dev-master": "3.0.x-dev"
        }
    },
    "scripts": {
        "test": "phpunit --testdox --no-interaction",
        "test-watch": "phpunit-watcher watch"
    },
    "config": {
        "sort-packages": true,
        "allow-plugins": {
            "infection/extension-installer": true,
            "composer/package-versions-deprecated": true
        }
    }
}
```


#### 包名称 <span id="package-name"></span>

每个 Composer 包都应该有一个包名称，该名称在所有其他包中唯一标识该包。包名称的格式是 `vendorName/projectName`。例如，在包名称 `yiisoft/queue` 中，供应商名称和项目名称分别是 `yiisoft` 和 `queue`。

> [!WARNING]
> 不要使用 `yiisoft` 作为你的供应商名称，因为它是为 Yii 本身保留的。

我们建议你为无法作为通用 PHP 包工作且需要 Yii 应用程序的包在项目名称前加上 `yii-` 前缀。这将使用户更容易判断包是否是 Yii 特定的。

#### 依赖项 <span id="dependencies"></span>

如果你的扩展依赖于其他包，你应该在 `composer.json` 的 `require` 部分列出它们。确保你还为每个依赖包列出适当的版本约束（例如 `^1.0`、`@stable`）。当你的扩展以稳定版本发布时，使用稳定的依赖项。

#### 类自动加载 <span id="class-autoloading"></span>

为了让你的类能够自动加载，你应该在 `composer.json` 文件中指定 `autoload` 条目，如下所示：

```json
{
    // ....

    "autoload": {
        "psr-4": {
            "MyVendorName\\MyPackageName\\": "src"
        }
    }
}
```

你可以列出一个或多个根命名空间及其对应的文件路径。

### 推荐实践 <span id="recommended-practices"></span>

因为包是供其他人使用的，所以在开发过程中你通常需要付出额外的努力。下面，我们介绍一些创建高质量扩展的常见和推荐实践。

#### 测试 <span id="testing"></span>

你希望你的包能够完美运行，而不会给其他人带来问题。为了达到这个目标，你应该在向公众发布之前测试你的扩展。

建议你创建各种测试用例来覆盖你的扩展代码，而不是依赖手动测试。每次在发布包的新版本之前，你可以运行这些测试用例以确保一切正常。有关更多详细信息，请参阅[测试](../testing/overview.md)部分。

#### 版本控制 <span id="versioning"></span>

你应该为扩展的每个版本提供一个版本号（例如 `1.0.1`）。我们建议你在确定应该使用什么版本号时遵循[语义化版本控制](https://semver.org)实践。

#### 发布 <span id="releasing"></span>

要让其他人知道你的包，你需要将其发布到公众。

如果这是你第一次发布包，你应该在 Composer 仓库（例如 [Packagist](https://packagist.org/)）中注册它。之后，你所需要做的就是在扩展的 VCS 仓库上创建一个发布标签（例如 `v1.0.1`）并通知 Composer 仓库有关新版本的信息。然后人们就能够通过 Composer 仓库找到新版本并安装或更新包。

在发布包时，除了代码文件之外，你还应该考虑包含以下内容以帮助其他人了解和使用你的扩展：

* 包根目录中的自述文件：它描述了你的扩展的功能以及如何安装和使用它。我们建议你使用 [Markdown](https://daringfireball.net/projects/markdown/) 格式编写并将文件命名为 `README.md`。
* 包根目录中的更新日志文件：它列出了每个版本中所做的更改。该文件可以使用 Markdown 格式编写并命名为 `CHANGELOG.md`。
* 包根目录中的升级文件：它提供了如何从扩展的旧版本升级的说明。该文件可以使用 Markdown 格式编写并命名为 `UPGRADE.md`。
* 教程、演示、截图等：如果你的扩展提供了许多无法在自述文件中完全涵盖的功能，则需要这些内容。
* API 文档：你的代码应该有良好的文档，以便其他人更容易阅读和理解它。
