# 响应

HTTP 响应有状态码和消息、一组头和一个主体：

```
HTTP/1.1 200 OK
Date: Mon, 27 Jul 2009 12:28:53 GMT
Server: Apache/2.2.14 (Win32)
Last-Modified: Wed, 22 Jul 2009 19:15:56 GMT
Content-Length: 6
Content-Type: text/html
Connection: Closed

Hello!
```

Yii 在 Web 应用程序中使用 [PSR-7 `Response`](https://www.php-fig.org/psr/psr-7/) 来表示响应。

该对象应该作为控制器操作或其他中间件执行的结果被构造和返回。
通常，中间件将响应工厂注入到其构造函数中。

```php
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

final readonly class PostAction
{
    public function __construct(
        private ResponseFactoryInterface $responseFactory
    )
    {
    }

    public function view(ServerRequestInterface $request): ResponseInterface
    {
        $response = $this->responseFactory->createResponse();
        $response->getBody()->write('Hello!');
        return $response;
    }
}
```

## 状态码

你可以像下面这样设置状态码：

```php
use Yiisoft\Http\Status;

$response = $response->withStatus(Status::NOT_FOUND);
```

为了方便和可读性，大多数状态码都可以从 `Status` 类中获取。

## 头

你可以像这样设置头：

```php
$response = $response->withHeader('Content-type', 'application/json');
```

如果需要将头值附加到现有头：

```php
$response = $response->withAddedHeader('Set-Cookie', 'qwerty=219ffwef9w0f; Domain=somecompany.co.uk; Path=/; Expires=Wed, 30 Aug 2019 00:00:00 GMT');
```

如果需要，可以删除头：

```php
$response = $response->withoutHeader('Set-Cookie');
```

## 主体

响应主体是实现 `Psr\Http\Message\StreamInterface` 的对象。

你可以通过接口本身写入它：

```php
$body = $response->getBody();
$body->write('Hello');
```


## 示例

### 重定向

```php
use Yiisoft\Http\Status;

return $response
  ->withStatus(Status::PERMANENT_REDIRECT)
  ->withHeader('Location', 'https://www.example.com');
```

请注意，重定向使用不同的状态：

| 代码 | 用法                          | 用途                                                                                                                                                          |
|------|------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 301  | `Status::MOVED_PERMANENTLY`  | 永久更改了 URL 结构。搜索引擎更新其索引，浏览器缓存它。                                                                                                        |
| 308  | `Status::PERMANENT_REDIRECT` | 类似于 301，但保证 HTTP 方法不会改变。                                                                                                                         |
| 302  | `Status::FOUND`              | 临时更改，如维护页面。原始 URL 仍应用于将来的请求。搜索引擎通常不会更新其索引。                                                                                 |
| 307  | `Status::TEMPORARY_REDIRECT` | 类似于 302，但保证 HTTP 方法不会改变。                                                                                                                         |
| 303  | `Status::SEE_OTHER`          | 在表单提交后防止用户刷新时重复提交。明确告诉使用 `GET` 进行重定向，即使原始请求是 `POST`。                                                                      |

### 使用 JSON 响应

```php
use Yiisoft\Http\Status;
use Yiisoft\Json\Json;

$data = [
    'account' => 'samdark',
    'value' => 42
];

$response->getBody()->write(Json::encode($data));
return $response
    ->withStatus(Status::OK)
    ->withHeader('Content-Type', 'application/json');
```
