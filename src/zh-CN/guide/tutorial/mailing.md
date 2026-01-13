# 邮件发送

Yii 使用 [yiisoft/mailer](https://github.com/yiisoft/mailer) 包简化了电子邮件的编写和发送。该包提供了内容组合功能和发送电子邮件的基本接口。默认情况下，该包包含一个文件邮件发送器，它将电子邮件内容写入文件而不是发送它们。这在应用开发的初始阶段特别有用。

要发送实际的电子邮件，您可以使用 [Symfony Mailer](https://github.com/yiisoft/mailer-symfony) 实现，下面的示例中使用的就是它。

## 配置邮件发送器

邮件发送器服务允许您创建消息实例、填充数据并发送它。通常，您从 DI 容器中获取一个 `Yiisoft\Mailer\MailerInterface` 实例。

您也可以手动创建一个实例，如下所示：

```php

use Yiisoft\Mailer\Symfony\Mailer;

/**
 * @var \Symfony\Component\Mailer\Transport\TransportInterface $transport
 */

$mailer = new \Yiisoft\Mailer\Symfony\Mailer(
    $transport,
);
```

`Yiisoft\Mailer\MailerInterface` 提供了两个主要方法：

- `send()` - 发送给定的电子邮件消息。
- `sendMultiple()` - 一次发送多条消息。

## 创建消息

### 简单文本消息

要创建一个带有文本正文的简单消息，使用 `Yiisoft\Mailer\Message`：

```php
$message = new \Yiisoft\Mailer\Message(
    from: 'from@domain.com',
    to: 'to@domain.com',
    subject: 'Message subject',
    textBody: 'Plain text content'
);
```

### 简单 HTML 消息

```php
$message = new \Yiisoft\Mailer\Message(
    from: 'from@domain.com',
    to: 'to@domain.com',
    subject: 'Message subject',
    htmlBody: '<b>HTML content</b>'
);
```

### 从模板创建 HTML 消息

在这个示例中，我们将使用渲染包 [view](https://github.com/yiisoft/view)。

```php
/**
 * @var \Yiisoft\View\View $view
 */

$content = $view->render('path/to/view.php', [
    'name' => 'name',
    'code' => 'code',
]);

$message = new \Yiisoft\Mailer\Message(
    from: 'from@domain.com',
    to: 'to@domain.com',
    subject: 'Subject',
    htmlBody: $content
);
```

### 使用布局

您还可以从模板消息向布局传递参数：

```php
/**
 * @var \Yiisoft\View\View $view
 * @var array $layoutParameters
 */

$messageBody = $view->render('path/to/view.php', [
    'name' => 'name',
    'code' => 'code',
]);

$layoutParameters['content'] = $messageBody;

$content = $view->render('path/to/layout.php', $layoutParameters);

$message = new \Yiisoft\Mailer\Message(
    from: 'from@domain.com',
    to: 'to@domain.com',
    subject: 'Subject',
    htmlBody: $content
);
```

### 布局示例

您可以将视图渲染结果包装在布局中，类似于 Web 应用中布局的工作方式。这对于设置共享内容（如 CSS 样式）很有用：

```php
<?php
/* @var $content string Mail contents as view render result */
?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
</head>
<body>

<?= $content ?>

<footer style="margin-top: 5em">
-- <br>
Mailed by Yii
</footer>

</body>
</html>
```

## 添加更多数据

`Yiisoft\Mailer\MessageInterface` 提供了几个方法来自定义您的消息：

- `withCharset()` - 返回一个具有指定字符集的新实例。
- `withFrom()` - 返回一个具有指定发件人电子邮件地址的新实例。
- `withTo()` - 返回一个具有指定收件人电子邮件地址的新实例。
- `withReplyTo()` - 返回一个具有指定回复地址的新实例。
- `withCc()` - 返回一个具有指定抄送（额外副本接收者）地址的新实例。
- `withBcc()` - 返回一个具有指定密送（隐藏副本接收者）地址的新实例。
- `withSubject()` - 返回一个具有指定消息主题的新实例。
- `withDate()` - 返回一个具有指定消息发送日期的新实例。
- `withPriority()` - 返回一个具有指定消息优先级的新实例。
- `withReturnPath()` - 返回一个具有指定返回路径（退信地址）的新实例。
- `withSender()` - 返回一个具有指定实际发件人电子邮件地址的新实例。
- `withHtmlBody()` - 返回一个具有指定消息 HTML 内容的新实例。
- `withTextBody()` - 返回一个具有指定消息纯文本内容的新实例。
- `withAddedHeader()` - 返回一个添加了指定自定义标头值的新实例。
- `withHeader()` - 返回一个具有指定自定义标头值的新实例。
- `withHeaders()` - 返回一个具有指定自定义标头值的新实例。

这些方法是不可变的，意味着它们返回一个带有更新数据的消息新实例。

注意 `with` 前缀。它表示该方法是不可变的，并返回一个具有更改数据的类的新实例。

### 获取器

以下获取器可用于检索消息数据：

- `getCharset()` - 返回此消息的字符集。
- `getFrom()` - 返回消息发件人电子邮件地址。
- `getTo()` - 返回消息收件人电子邮件地址。
- `getReplyTo()` - 返回此消息的回复地址。
- `getCc()` - 返回此消息的抄送（额外副本接收者）地址。
- `getBcc()` - 返回此消息的密送（隐藏副本接收者）地址。
- `getSubject()` - 返回消息主题。
- `getDate()` - 返回消息发送日期，如果未设置则返回 null。
- `getPriority()` - 返回此消息的优先级。
- `getReturnPath()` - 返回此消息的返回路径（退信地址）。
- `getSender()` - 返回消息实际发件人电子邮件地址。
- `getHtmlBody()` - 返回消息 HTML 正文。
- `getTextBody()` - 返回消息文本正文。
- `getHeader()` - 返回指定标头的所有值。
- `__toString()` - 返回此消息的字符串表示形式。

## 附加文件

您可以使用 `withAttached()` 方法将文件附加到消息中：

```php
use Yiisoft\Mailer\File;

// 从本地文件系统附加文件
$message = $message->withAttached(
    File::fromPath('/path/to/source/file.pdf'),
);

// 动态创建附件
$message = $message->withAttached(
    File::fromContent('Attachment content', 'attach.txt', 'text/plain'),
);
```

## 嵌入图片

您可以使用 `withEmbedded()` 方法将图片嵌入到消息内容中。这在使用视图编写消息时特别有用：

```php
$logo = 'path/to/logo';
$htmlBody = $this->view->render(
    __DIR__ . 'template.php',
    [
        'content' => $content,
        'logoCid' => $logo->cid(),
    ],
);
return new \Yiisoft\Mailer\Message(
            from: 'from@domain.com',
            to: 'to@domain.com',
            subject: 'Message subject',
            htmlBody: $htmlBody,
            embeddings: $logo
        );
```

在您的视图或布局模板中，您可以使用其 CID 引用嵌入的图片：

```php
<img src="<?= $logoCid; ?>">
```

## 发送消息

要发送电子邮件消息：

```php
/**
 * @var \Yiisoft\View\View $view
 */

$content = $view->render('path/to/view.php', [
    'name' => 'name',
    'code' => 'code',
]);

$message = new \Yiisoft\Mailer\Message(
    from: 'from@domain.com',
    to: 'to@domain.com',
    subject: 'Subject',
    htmlBody: $content
);

$mailer->send($message);
```

## 发送多条消息

您可以一次发送多条消息：

```php
$messages = [];

foreach ($users as $user) {
    $messages[] = (new \Yiisoft\Mailer\Message())
        // ...
        ->withTo($user->email);
}

$result = $mailer->sendMultiple($messages);
```

`sendMultiple()` 方法返回一个 `Yiisoft\Mailer\SendResults` 对象，其中包含成功发送和失败消息的数组。

## 实现您自己的邮件驱动程序

要创建自定义邮件解决方案，请实现 `Yiisoft\Mailer\MailerInterface` 和 `Yiisoft\Mailer\MessageInterface` 接口。

## 用于开发

对于本地或测试开发，您可以使用不发送电子邮件的简化邮件发送器实现。该包提供了这些实现：

- `Yiisoft\Mailer\StubMailer` - 一个将消息存储在本地数组中的简单邮件发送器。
- `Yiisoft\Mailer\FileMailer` - 一个将电子邮件消息保存为文件而不是发送它们的模拟邮件发送器。
- `Yiisoft\Mailer\NullMailer` - 一个丢弃消息而不发送或存储它们的邮件发送器。

要使用这些邮件发送器之一，请在您的开发环境文件中配置它，例如：`environments/local/di.php`

```php
return [
    Yiisoft\Mailer\MailerInterface::class => Yiisoft\Mailer\StubMailer::class, //or any other
];

```
