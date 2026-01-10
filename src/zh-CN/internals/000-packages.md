# 000 — 包

Yii3 团队将框架划分为多个包，这些包遵循以下约定。

对于所有包，GitHub 仓库名称与 Packagist 包名称完全匹配。

有关包的完整列表及其构建状态，请参阅 [yiiframework.com 的状态页面](https://www.yiiframework.com/status/3.0)。

## Yii 专用包（框架和扩展）
    
- 命名为 `yiisoft/yii-something` 或更具体的：`yii-type-something`，例如：
    - 模块：`yii-module-users`、`yii-module-pages`
    - 主题：`yii-theme-adminlte`、`yii-theme-hyde`
    - 小部件：`yii-widget-datepicker`
    - ...
- 标题为 "Yii Framework ..."
- 可以有任何依赖项和 Yii 专用代码

## 通用包（库）
  
- 你可以独立于 Yii 框架使用这些包
- 命名为 `yiisoft/something`，不带 yii 前缀
- 标题为 "Yii ..."
- 不得依赖任何 Yii 专用包
- 应该尽可能少的依赖项

## 配置和默认值

以下内容适用于 Yii 专用包和通用包：

- 包可以有 `config` 目录，其中包含 Yii 专用的默认值。
- 包可以在 `composer.json` 的 "extra" 部分中包含 "config-plugin"。
- 包不得在 `composer.json` 的 `require` 部分中包含仅在 `config` 中使用的依赖项。
- 你应该使用 `vendor/package-name` 命名空间参数：

```php
return [
    'vendor/package-name' => [
        'param1' => 1,
        'param2' => 2,
    ],
];
```
  
## 版本

所有包都遵循 [SemVer](https://semver.org/) 版本控制：

- `x.*.*` - 不兼容的 API 更改。
- `*.x.*` - 添加功能（向后兼容）。
- `*.*.x` - 错误修复（向后兼容）。

第一个稳定版本应该是 1.0.0。

每个包的版本号不依赖于任何其他包的版本或框架名称/版本，仅依赖于其自身的公共契约。
整个框架的名称为 "Yii3"。

只要兼容，就可以一起使用不同主版本的包。

## PHP 版本支持

包支持的 PHP 版本取决于 [PHP 版本生命周期](https://www.php.net/supported-versions.php)。

- 具有活跃支持的包版本必须支持所有具有活跃支持的 PHP 版本。
- 包和应用程序模板都必须有接收错误和安全修复的受支持版本。
  这些应该对应于接收安全修复的 PHP 版本。
- 包和应用程序模板可能有适用于不受支持的 PHP 版本的受支持版本。
- 在包或应用程序模板中提升最低 PHP 版本是一个次要更改。

## composer.json

版本范围中的逻辑 OR 运算符必须使用双管道符号（`||`）。例如：`"yiisoft/arrays": "^1.0 || ^2.0"`。
