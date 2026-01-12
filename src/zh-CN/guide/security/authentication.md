# 身份验证

身份验证是验证用户身份的过程。它通常使用标识符（例如用户名或电子邮件地址）和 secret 令牌（例如 password 或访问令牌）来判断用户是否是其声称的人。身份验证是登录功能的基础。

Yii 提供了一个身份验证框架,它连接各种组件以支持登录。要使用此框架,你主要需要执行以下工作:

* 配置 `Yiisoft\Yii\Web\User\User` 服务;
* 创建一个实现 `\Yiisoft\Auth\IdentityInterface` 接口的类;
* 创建一个实现 `\Yiisoft\Auth\IdentityRepositoryInterface` 接口的类;

## 配置 `Yiisoft\Yii\Web\User\User` <span id="configuring-user"></span>

`Yiisoft\Yii\Web\User\User` 应用程序服务管理用户身份验证状态。它依赖于 `Yiisoft\Auth\IdentityRepositoryInterface`,该接口应返回具有实际身份验证逻辑的 `\Yiisoft\Auth\IdentityInterface` 实例。

```php
use Yiisoft\Session\Session;
use Yiisoft\Session\SessionInterface;
use Yiisoft\Auth\IdentityRepositoryInterface;
use Psr\Container\ContainerInterface;

return [
    // ...

    SessionInterface::class => [
        'class' => Session::class,
        '__construct()' => [
            $params['session']['options'] ?? [],
            $params['session']['handler'] ?? null,
        ],
    ],
    
    
    // User:
    IdentityRepositoryInterface::class => static function (ContainerInterface $container) {
        // 你可以使用任何实现来代替基于 Cycle 的仓库
        return $container->get(\Cycle\ORM\ORMInterface::class)->getRepository(\App\Entity\User::class);
    },
];
```

## 实现 `\Yiisoft\Auth\IdentityInterface` <span id="implementing-identity"></span>

身份类必须实现 `\Yiisoft\Auth\IdentityInterface` 接口,该接口只有一个方法:

* [[yii\web\IdentityInterface::getId()|getId()]]: 它返回此身份实例所代表的用户的 ID。

在以下示例中,身份类被实现为纯 PHP 对象。

```php
<?php

namespace App\User;

use Yiisoft\Auth\IdentityInterface;

final readonly class Identity implements IdentityInterface
{
    public function __construct(
        private string $id
    ) {
    }

    public function getId(): string
    {
        return $this->id;
    }
}
```

## 实现 `\Yiisoft\Auth\IdentityRepositoryInterface` <span id="implementing-identity-repository"></span>

身份仓库类必须实现 `\Yiisoft\Auth\IdentityRepositoryInterface` 接口,该接口具有以下方法:

* `findIdentity(string $id): ?IdentityInterface`: 它使用指定的 ID 查找身份类的实例。当你需要通过会话保持登录状态时使用此方法。
* `findIdentityByToken(string $token, string $type): ?IdentityInterface`: 它使用指定的访问令牌查找身份类的实例。当你需要通过单个 secret 令牌对用户进行身份验证时使用此方法(例如,在无状态 REST API 中)。
  
一个虚拟实现可能如下所示:

```php
namespace App\User;

use App\User\Identity;
use \Yiisoft\Auth\IdentityInterface;
use \Yiisoft\Auth\IdentityRepositoryInterface;

final readonly class IdentityRepository implements IdentityRepositoryInterface
{
    private const USERS = [
      [
        'id' => 1,
        'token' => '12345'   
      ],
      [
        'id' => 42,
        'token' => '54321'
      ],  
    ];

    public function findIdentity(string $id) : ?IdentityInterface
    {
        foreach (self::USERS as $user) {
            if ((string)$user['id'] === $id) {
                return new Identity($id);            
            }
        }
        
        return null;
    }

    public function findIdentityByToken(string $token, string $type) : ?IdentityInterface
    {
        foreach (self::USERS as $user) {
             if ($user['token'] === $token) {
                 return new Identity((string)$user['id']);            
             }
         }
         
         return null;
    }
}
```

## 使用 `\Yiisoft\User\User` <span id="using-user"></span>

你可以使用 `\Yiisoft\User\User` 服务来获取当前用户身份。
作为任何服务,它可以在操作处理程序构造函数或方法中自动注入:

```php
use \Psr\Http\Message\ServerRequestInterface;
use \Yiisoft\User\User;

final readonly class SiteController
{
    public function actionIndex(ServerRequestInterface $request, User $user)
    if ($user->isGuest()) {
        // 用户是访客
    } else {
        $identity = $user->getIdentity();
        // 根据身份执行某些操作
    }
        }        
    }
}
```

`isGuest()` 确定用户是否已登录。`getIdentity()` 返回身份实例。

要登录用户,你可以使用以下代码:

```php
$identity = $identityRepository->findByEmail($email);

/* @var $user \Yiisoft\User\User */
$user->login($identity);
```

`login()` 方法将身份设置到 User 服务。
它将身份存储到会话中,以便维护用户身份验证状态。

要注销用户,调用

```php
/* @var $user \Yiisoft\User\User */
$user->logout();
```

## 身份验证事件 <span id="auth-events"></span>

用户服务在登录和注销过程中会触发一些事件。


* `\Yiisoft\User\Event\BeforeLogin`: 在 `login()` 开始时触发。
  如果事件处理程序在事件对象上调用 `invalidate()`,登录过程将被取消。
* `\Yiisoft\User\Event\AfterLogin`: 在成功登录后触发。
* `\Yiisoft\User\Event\BeforeLogout`: 在 `logout()` 开始时触发。
  如果事件处理程序在事件对象上调用 `invalidate()`,注销过程将被取消。
* `\Yiisoft\User\Event\AfterLogout`: 在成功注销后触发。

你可以响应这些事件来实现登录审计、在线用户统计等功能。例如,在 `\Yiisoft\User\Event\AfterLogin` 的处理程序中,你可以在 `user` 数据库表中记录登录时间和 IP 地址。
