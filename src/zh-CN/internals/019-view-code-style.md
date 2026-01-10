# 019 — 视图代码风格

视图文件中的 PHP 代码不应该复杂。
代码必须包含负责格式化数据的逻辑，但不包含请求此数据的逻辑。

## 头部

视图文件头部用于放置描述可用变量的 phpdoc 和导入类：

\`\`\`php
<?php

declare(strict_types=1);

/** @var Post $post */
/** @var string $name */

use Yiisoft\Html\Html;
\`\`\`

## 控制结构

首选控制结构的替代语法，如 `foreach` 和 `if`：

\`\`\`php
<?php foreach ($posts as $post): ?>   
    <h2><?= Html::encode($post->getTitle()) ?></h2>
    <p><?= Html::encode($post->getDescription()) ?></p>
<?php endforeach; ?>
\`\`\`

## 短回显

首选短回显：

\`\`\`php
<?= Html::encode($name) ?>
\`\`\`

## 类方法

视图文件中使用的所有类方法都必须是公共的，无论视图是否由类本身渲染。
