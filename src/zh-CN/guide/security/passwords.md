# 使用 passwords

大多数开发人员都知道 passwords 不能以纯文本形式存储,但许多开发人员认为使用 `md5`、`sha1` 或 `sha256` 等对 passwords 进行哈希处理仍然是安全的。曾经有一段时间使用上述哈希算法就足够了,但现代硬件使得可以使用暴力攻击来逆向这些哈希甚至更强的哈希。

为了为用户 passwords 提供更高的安全性,即使在最坏的情况下(当有人破坏你的应用程序时),你需要使用能够抵御暴力攻击的哈希算法。
目前最好的选择是 `argon2`。
Yii `yiisoft/security` 包使安全生成和验证哈希变得更容易,并确保使用最佳的哈希解决方案。

要使用它,你需要首先安装该包:

```
composer require yiisoft/security
```

当用户第一次提供 password 时(例如,注册时),需要对 password 进行哈希处理并存储:


```php
$hash = (new PasswordHasher())->hash($password);

// 将哈希保存到数据库或其他存储
saveHash($hash);
```

当用户尝试登录时,必须根据先前哈希和存储的 password 验证提交的 password:


```php
// 从数据库或其他存储获取哈希
$hash = getHash();

if ((new PasswordHasher())->validate($password, $hash)) {
    // 一切正常,登录用户
} else {
    // 错误的 password
}
```
