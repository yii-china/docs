# 模板引擎

Yii3 通过灵活的渲染器系统支持多种模板引擎。默认情况下，PHP 被用作模板引擎，但你可以轻松添加对其他引擎（如 Twig）的支持，或创建自己的自定义渲染器。

PHP 模板在"[视图](view.md)"指南章节中已有描述。

## Twig 模板引擎

Twig 是一个现代化的模板引擎，提供了更友好的设计师语法。要在你的 Yii3 应用程序中使用 Twig，你需要安装 Twig 扩展。

```bash
composer require yiisoft/view-twig
```

现在你可以使用 `.twig` 模板了。例如，`views/site/about.twig`：

```twig
{# IDE 支持的变量类型提示 #}
{# @var user \App\Entity\User #}
{# @var posts \App\Entity\Post[] #}

<div class="user-profile">
    <h1>{{ user.name }}</h1>
    <p>邮箱: {{ user.email }}</p>
    
    {% if posts is not empty %}
        <h2>最近的文章</h2>
        <ul class="posts">
            {% for post in posts %}
                <li>
                    <h3>{{ post.title }}</h3>
                    <p>{{ post.excerpt }}</p>
                    <time>{{ post.publishedAt|date('F j, Y') }}</time>
                </li>
            {% endfor %}
        </ul>
    {% else %}
        <p>暂无文章。</p>
    {% endif %}
</div>
```

## Twig 特性

**自动转义**：Twig 会自动转义 HTML 上下文中的变量：

```twig
{# 自动转义 #}
<h1>{{ title }}</h1>

{# 原始输出（谨慎使用） #}
<div>{{ content|raw }}</div>
```

**过滤器和函数**：Twig 提供了许多内置的过滤器和函数：

```twig
{# 日期格式化 #}
<time>{{ post.createdAt|date('Y-m-d H:i') }}</time>

{# 字符串操作 #}
<p>{{ description|truncate(100) }}</p>

{# URL 生成 #}
<a href="{{ path('user.profile', {'id': user.id}) }}">个人资料</a>
```

**模板继承**：Twig 支持模板继承：

**views/layout/main.twig**
```twig
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}默认标题{% endblock %}</title>
</head>
<body>
    <main>
        {% block content %}{% endblock %}
    </main>
</body>
</html>
```

**views/site/about.twig**
```twig
{% extends "layout/main.twig" %}

{% block title %}关于我们{% endblock %}

{% block content %}
    <h1>关于我们公司</h1>
    <p>欢迎访问我们的网站！</p>
{% endblock %}
```

### 渲染 Twig 模板

像使用 PHP 模板一样使用 Twig 模板：
```php
// 在你的控制器中
public function about(): ResponseInterface
{
    return $this->viewRenderer->render('about.twig', [
        'user' => $this->getCurrentUser(),
        'posts' => $this->getRecentPosts(),
    ]);
}
```

## 自定义模板引擎

你可以通过实现 `TemplateRendererInterface` 来创建自定义模板引擎：

```php
<?php

declare(strict_types=1);

namespace App\View;

use Yiisoft\View\TemplateRendererInterface;

final class MarkdownRenderer implements TemplateRendererInterface
{
    public function __construct(
        private MarkdownParserInterface $parser,
    ) {}

    public function render(string $template, array $parameters = []): string
    {
        $content = file_get_contents($template);
        
        // 用参数替换占位符
        foreach ($parameters as $key => $value) {
            $content = str_replace("{{$key}}", (string) $value, $content);
        }
        
        return $this->parser->parse($content);
    }
}
```

注册你的自定义渲染器：

```php
use Yiisoft\Container\Reference;

// 在配置中
'yiisoft/view' => [
    'renderers' => [
        'md' => Reference::to(App\View\MarkdownRenderer::class),
    ],
],
```

现在你可以使用 `.md` 模板文件了：

**views/content/help.md**
```markdown
# 帮助: {{title}}

欢迎, {{username}}!

这是一个 Markdown 模板，包含 **粗体** 和 *斜体* 文本。

- 功能 1
- 功能 2
- 功能 3
```

## 选择合适的模板引擎

**使用 PHP 模板的情况：**
- 你需要最大的灵活性和性能
- 你的团队熟悉 PHP
- 你想利用现有的 PHP 知识
- 你需要在模板中使用复杂逻辑（尽管应该尽量减少）

**使用 Twig 模板的情况：**
- 你想要更严格地分离逻辑和表现
- 你与喜欢更简洁语法的设计师合作
- 你需要自动转义和安全特性
- 你想要模板继承和高级功能

**使用自定义模板的情况：**
- 你有 PHP 或 Twig 无法满足的特定需求
- 你正在处理专门的内容格式
- 你需要与外部模板系统集成

## 最佳实践

1. **保持模板简单**：将复杂逻辑移至控制器或服务
2. **始终转义输出**：通过正确转义变量来防止 XSS 攻击
3. **使用有意义的名称**：清晰地命名你的模板和变量
4. **组织模板**：将相关模板分组到子目录中
5. **记录变量**：始终添加类型提示以获得更好的 IDE 支持
6. **避免业务逻辑**：将业务逻辑保留在模型和服务中
