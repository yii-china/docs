# 术语表

# A

## 别名（alias）

别名是 Yii 用来引用类或目录的字符串,例如 `@app/vendor`。
更多信息请阅读 ["别名"](concept/aliases.md)。

## 资源（asset）

资源是指资源文件。通常包含 JavaScript 或 CSS 代码,但也可以是通过 HTTP 访问的任何静态内容。
更多信息请阅读 ["资源"](views/asset.md)。

# C

## 配置（configuration）

配置可以指设置对象属性的过程,也可以指存储对象或对象类设置的配置文件。
更多信息请阅读 ["配置"](concept/configuration.md)。

# D

## 依赖注入（DI）

依赖注入是一种编程技术,其中一个对象注入一个依赖对象。
更多信息请阅读 ["依赖注入和容器"](concept/di-container.md)。

# I

## 安装（installation）

安装是通过遵循 readme 文件或执行特别准备的脚本来准备某些东西工作的过程。在 Yii 的情况下,它是设置权限和满足软件要求。

# M

## 中间件（middleware）

中间件是请求处理堆栈中的处理器。给定一个请求,它可以产生响应或执行某些操作并将处理传递给下一个中间件。
更多信息请阅读 ["中间件"](structure/middleware.md)。

## 模块（module）

模块是基于用例对某些代码进行分组的命名空间。它通常在主应用程序中使用,可以包含任何源代码、定义额外的 URL 处理器或控制台命令。

# N

## 命名空间（namespace）

命名空间是指 [PHP 语言特性](https://www.php.net/manual/en/language.namespaces.php),用于将多个类分组到某个名称下。

# P

## 包（package）

包通常是指 [Composer 包](https://getcomposer.org/doc/)。它是可重用和可重新分发的代码,可通过包管理器自动安装。

# Q

## 队列（queue）

队列类似于堆栈。队列遵循先进先出(First-In-First-Out)方法。
Yii 有一个 [yiisoft/queue](https://github.com/yiisoft/queue) 包。

# R

## 规则（rule）

规则通常是指 [yiisoft/validator](https://github.com/yiisoft/validator) 包的验证规则。
它包含一组用于检查数据集是否有效的参数。
"规则处理器"执行实际处理。

# V

## 供应商（vendor）

供应商是以包的形式提供代码的组织或个人开发者。
它也可以指 [Composer 的 `vendor` 目录](https://getcomposer.org/doc/)。
