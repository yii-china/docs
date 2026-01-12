# Cookies

Cookies 用于通过使用 HTTP 头将数据发送到客户端浏览器来在请求之间持久化数据。
客户端在请求头中将数据发送回服务器。因此，cookies 便于存储少量数据，例如令牌或标志。

## 读取 cookies

你可以从服务器请求中获取 Cookie 值，该请求可作为路由处理程序（例如控制器操作）参数使用：

```php
private function actionProfile(\Psr\Http\Message\ServerRequestInterface $request)
{
    $cookieValues = $request->getCookieParams();
    $cookieValue = $cookieValues['cookieName'] ?? null;
    // ...
}
```

除了直接从服务器请求获取 cookie 值外，你还可以使用 [yiisoft/request-provider](https://github.com/yiisoft/request-provider) 包，它通过 `\Yiisoft\RequestProvider\RequestCookieProvider` 提供了一种更结构化的方式来处理 cookies。
这种方法可以简化你的代码并提高可读性。

以下是使用 `\Yiisoft\RequestProvider\RequestCookieProvider` 处理 cookies 的示例：

```php
final readonly class MyService
{
    public function __construct(
        private \Yiisoft\RequestProvider\RequestCookieProvider $cookies
    ) {}

    public function go(): void
    {
        // 检查特定 cookie 是否存在
        if ($this->cookies->has('foo')) {
            // 检索 cookie 的值
            $fooValue = $this->cookies->get('foo');
            // 对 cookie 值执行某些操作
        }

        // 检索另一个 cookie 值
        $barValue = $this->cookies->get('bar');
        // 对 bar cookie 值执行某些操作
    }
}

## 发送 cookies

由于发送 cookies 实际上是发送头，但由于形成头并不简单，因此有 `\Yiisoft\Cookies\Cookie` 类来帮助实现：

```php
$cookie = (new \Yiisoft\Cookies\Cookie('cookieName', 'value'))
    ->withPath('/')
    ->withDomain('yiiframework.com')
    ->withHttpOnly(true)
    ->withSecure(true)
    ->withSameSite(\Yiisoft\Cookies\Cookie::SAME_SITE_STRICT)
    ->withMaxAge(new \DateInterval('P7D'));

return $cookie->addToResponse($response);
```

形成 cookie 后，调用 `addToResponse()` 并传递 `\Psr\Http\Message\ResponseInterface` 的实例以向其添加相应的 HTTP 头。

## 签名和加密 cookies

为了防止 cookie 值被替换，该包提供了两种实现：

`Yiisoft\Cookies\CookieSigner` - 使用基于 cookie 值和密钥的唯一前缀哈希对每个 cookie 进行签名。
`Yiisoft\Cookies\CookieEncryptor` - 使用密钥加密每个 cookie。

加密比签名更安全，但性能较低。

```php
$cookie = new \Yiisoft\Cookies\Cookie('identity', 'identityValue');

// 用于签名和验证 cookies 的密钥。
$key = '0my1xVkjCJnD_q1yr6lUxcAdpDlTMwiU';

$signer = new \Yiisoft\Cookies\CookieSigner($key);
$encryptor = new \Yiisoft\Cookies\CookieEncryptor($key);

$signedCookie = $signer->sign($cookie);
$encryptedCookie = $encryptor->encrypt($cookie);
```

要验证并获取纯值，请使用 `validate()` 和 `decrypt()` 方法。

```php
$cookie = $signer->validate($signedCookie);
$cookie = $encryptor->decrypt($encryptedCookie);
```

如果 cookie 值被篡改或之前未被签名/加密，将抛出 `\RuntimeException`。
因此，如果你不确定 cookie 值之前是否被签名/加密，请分别先使用 `isSigned()` 和 `isEncrypted()` 方法。

```php
if ($signer->isSigned($cookie)) {
    $cookie = $signer->validate($cookie);
}

if ($encryptor->isEncrypted($cookie)) {
    $cookie = $encryptor->decrypt($cookie);
}
```

如果你存储用户不应更改的重要数据，则对 cookie 的值进行签名或加密是有意义的。

### 自动化加密和签名

要自动化 cookie 值的加密/签名和解密/验证，请使用 `Yiisoft\Cookies\CookieMiddleware` 的实例，它是 [PSR-15](https://www.php-fig.org/psr/psr-15/) 中间件。

此中间件提供以下功能：

- 验证和解密来自请求的 cookie 参数值。
- 如果 cookie 参数被篡改，则从请求中排除该参数并记录相关信息。
- 加密/签名 cookie 值并在响应的 `Set-Cookie` 头中替换其纯值。

为了让中间件知道哪些 cookies 的哪些值需要加密/签名，必须将设置数组传递给其构造函数。数组键是 cookie 名称模式，值是 `CookieMiddleware::ENCRYPT` 或 `CookieMiddleware::SIGN` 的常量值。

```php
use Yiisoft\Cookies\CookieMiddleware;

$cookiesSettings = [
    // 与名称 `identity` 完全匹配。
    'identity' => CookieMiddleware::ENCRYPT,
    // 匹配下划线后的任何 1 到 9 的数字。
    'name_[1-9]' => CookieMiddleware::SIGN,
    // 匹配前缀后的任何字符串，包括空字符串，
    // 但不包括分隔符 "/" 和 "\"。
    'prefix*' => CookieMiddleware::SIGN,
];
```

有关使用通配符模式的更多信息，请参阅 [yiisoft/strings](https://github.com/yiisoft/strings#wildcardpattern-usage) 包。

创建和使用中间件：

```php
/**
 * @var \Psr\Http\Message\ServerRequestInterface $request
 * @var \Psr\Http\Server\RequestHandlerInterface $handler
 * @var \Psr\Log\LoggerInterface $logger
 */

// 用于签名和验证 cookies 的密钥。
$key = '0my1xVkjCJnD_q1yr6lUxcAdpDlTMwiU';
$signer = new \Yiisoft\Cookies\CookieSigner($key);
$encryptor = new \Yiisoft\Cookies\CookieEncryptor($key);

$cookiesSettings = [
    'identity' => \Yiisoft\Cookies\CookieMiddleware::ENCRYPT,
    'session' => \Yiisoft\Cookies\CookieMiddleware::SIGN,
];

$middleware = new \Yiisoft\Cookies\CookieMiddleware(
    $logger
    $encryptor,
    $signer,
    $cookiesSettings,
);

// 来自请求的 cookie 参数值被解密/验证。
// cookie 值被加密/签名并附加到响应中。
$response = $middleware->process($request, $handler);
```

如果 `$cookiesSettings` 数组为空，则不会加密和签名任何 cookies。

## Cookies 安全

你应该将每个 cookie 配置为安全的。重要的安全设置有：

- `httpOnly`。将其设置为 `true` 将阻止 JavaScript 访问 cookie 值。
- `secure`。将其设置为 `true` 将阻止通过 `HTTP` 发送 cookie。它将仅通过 `HTTPS` 发送。
- `sameSite`，如果设置为 `SAME_SITE_LAX` 或 `SAME_SITE_STRICT`，将阻止在跨站点浏览上下文中发送 cookie。`SAME_SITE_LAX` 将在容易受 CSRF 攻击的请求方法（例如 POST、PUT、PATCH 等）期间阻止 cookie 发送。`SAME_SITE_STRICT` 将阻止所有方法的 cookies 发送。
- 如果值中的数据不应被篡改，请对 cookie 的值进行签名或加密以防止值欺骗。
