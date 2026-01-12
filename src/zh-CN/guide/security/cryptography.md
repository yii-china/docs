# 密码学

在本节中,我们将回顾以下安全方面:

- 生成随机数据
- 加密和解密
- 确认数据完整性

要使用这些功能,你需要安装 `yiisoft/security` 包:

```
composer install yiisoft/security
```

## 生成伪随机数据

伪随机数据在许多情况下都很有用。例如,当通过电子邮件重置 password 时,你需要生成一个令牌,将其保存到数据库,并通过电子邮件发送给最终用户,这反过来将允许他们证明该帐户的所有权。重要的是,此令牌必须是唯一的且难以猜测,否则攻击者可能会预测令牌的值并重置用户的 password。

`\Yiisoft\Security\Random` 使生成伪随机数据变得简单:

```php
$key = \Yiisoft\Security\Random::string(42);
```

上面的代码将为你提供一个由 42 个字符组成的随机字符串。

如果你需要字节或整数,请直接使用 PHP 函数:

- `random_bytes()` 用于字节。请注意,输出可能不是 ASCII。
- `random_int()` 用于整数。

## 加密和解密

Yii 提供了方便的辅助函数来使用 secret 密钥加密/解密数据。
数据通过加密函数传递,以便只有拥有 secret 密钥的人才能解密它。
例如,你需要在数据库中存储一些信息,但你需要确保只有拥有 secret 密钥的用户才能查看它(即使有人破坏了应用程序数据库):

```php
$encryptedData = (new \Yiisoft\Security\Crypt())->encryptByPassword($data, $password);

// 将数据保存到数据库或其他存储
saveData($encryptedData);
```

解密它:

```php
// 从数据库或其他存储收集加密数据
$encryptedData = getEncryptedData();

$data = (new \Yiisoft\Security\Crypt())->decryptByPassword($encryptedData, $password);
```

你可以使用密钥而不是 password:

```php
$encryptedData = (new \Yiisoft\Security\Crypt())->encryptByKey($data, $key);

// 将数据保存到数据库或其他存储
saveData($encryptedData);
```

解密它:

```php
// 从数据库或其他存储收集加密数据
$encryptedData = getEncryptedData();

$data = (new \Yiisoft\Security\Crypt())->decryptByKey($encryptedData, $key);
```

## 确认数据完整性

在某些情况下,你需要验证你的数据没有被第三方篡改或以某种方式损坏。Yii 提供了一种通过 MAC 签名确认数据完整性的方法。

`$key` 应该在发送和接收双方都存在。在发送方:

```php
$signedMessage = (new \Yiisoft\Security\Mac())->sign($message, $key);

sendMessage($signedMessage);
```

在接收方:

```php
$signedMessage = receiveMessage($signedMessage);

try {
    $message = (new \Yiisoft\Security\Mac())->getMessage($signedMessage, $key);
} catch (\Yiisoft\Security\DataIsTamperedException $e) {
    // 数据被篡改
}
```

## 掩码令牌长度

掩码令牌通过随机化每个请求上输出令牌的方式来帮助缓解 BREACH 攻击。
对令牌应用随机掩码,使字符串始终唯一。

要掩码令牌:

```php
$maskedToken = \Yiisoft\Security\TokenMask::apply($token);
```

要从掩码中获取原始值:

```php
$token = \Yiisoft\Security\TokenMask::remove($maskedToken);
```
