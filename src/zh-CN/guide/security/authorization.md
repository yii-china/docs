# 授权

授权是验证用户是否有足够权限执行某项操作的过程。

## 检查权限 <span id="checking-for-permission"></span>

你可以使用 `\Yiisoft\User\User` 服务检查用户是否具有某些权限:

```php
namespace App\Blog\Post;

use Yiisoft\Router\CurrentRoute;
use Yiisoft\User\User;

final readonly class PostController
{
    public function actionEdit(CurrentRoute $route, User $user, PostRepository $postRepository)
    {
        $postId = $route->getArgument('id');
        if ($postId === null) {
            // 响应 404
        }
        
        $post = $postRepository->findByPK($postId);
        if ($post === null) {
            // 响应 404
        }

        if (!$this->canEditPost($user, $post)) {
            // 响应 403
        }
        
        // 继续编辑文章
    }
    
    private function canEditPost(User $user, Post $post): bool
    {
        return $post->getAuthorId() === $user->getId() || $user->can('updatePost');    
    }
}
```

在幕后,`Yiisoft\Yii\Web\User\User::can()` 方法调用 `\Yiisoft\Access\AccessCheckerInterface::userHasPermission()`,因此你应该在依赖容器中提供一个实现才能使其工作。

## 基于角色的访问控制 (RBAC) <span id="rbac"></span>

基于角色的访问控制 (RBAC) 提供了一种简单而强大的集中式访问控制。有关 RBAC 与其他更传统的访问控制方案的比较详情,请参阅 [Wikipedia](https://en.wikipedia.org/wiki/Role-based_access_control)。

Yii 实现了通用分层 RBAC,遵循 [NIST RBAC 模型](https://csrc.nist.gov/CSRC/media/Publications/conference-paper/2000/07/26/the-nist-model-for-role-based-access-control-towards-a-unified-/documents/sandhu-ferraiolo-kuhn-00.pdf)。

使用 RBAC 涉及两部分工作。第一部分是构建 RBAC 授权数据,第二部分是在必要的地方使用授权数据执行访问检查。由于 RBAC 实现了 `\Yiisoft\Access\AccessCheckerInterface`,使用它类似于使用访问检查器的任何其他实现。

为了便于接下来的描述,首先介绍一些基本的 RBAC 概念。

### 基本概念 <span id="basic-concepts"></span>

角色代表*权限*的集合(例如,创建文章、更新文章)。
你可以将角色分配给一个或多个用户。
要检查用户是否具有指定的权限,你可以检查用户是否具有该权限的角色。

与每个角色或权限关联的可能有一个*规则*。
规则代表访问检查器将执行的一段代码,以决定相应的角色或权限是否适用于当前用户。
例如,"更新文章"权限可能有一个规则来检查当前用户是否是文章创建者。
在访问检查期间,如果用户不是文章创建者,则没有"更新文章"权限。

角色和权限都处于层次结构中。
特别是,一个角色可能由其他角色或权限组成。
一个权限可能由其他权限组成。
Yii 实现了*偏序*层次结构,其中包括更特殊的*树*层次结构。
虽然角色可以包含权限,但反之则不成立。

### 配置 RBAC <span id="configuring-rbac"></span>

RBAC 通过 `yiisoft/rbac` 包提供,因此你需要安装它:

```
composer require yiisoft/rbac
```

在开始定义授权数据和执行访问检查之前,你需要在依赖容器中配置 `\Yiisoft\Access\AccessCheckerInterface`:

```php
use \Psr\Container\ContainerInterface;
use Yiisoft\Rbac\Manager\PhpManager;
use Yiisoft\Rbac\RuleFactory\ClassNameRuleFactory;

return [
    \Yiisoft\Access\AccessCheckerInterface::class => static function (ContainerInterface $container) {
        $aliases = $container->get(\Yiisoft\Aliases\Aliases::class);
        return new PhpManager(new ClassNameRuleFactory(), $aliases->get('@rbac'));
    }
];
```

`\Yiisoft\Rbac\Manager\PhpManager` 使用 PHP 脚本文件存储授权数据。
这些文件位于 `@rbac` 别名下。
如果你想在线更改权限层次结构,请确保该目录及其中的所有文件对 Web 服务器进程可写。

### 构建授权数据 <span id="generating-rbac-data"></span>

构建授权数据涉及以下任务:

- 定义角色和权限;
- 建立角色和权限之间的关系;
- 定义规则;
- 将规则与角色和权限关联;
- 将角色分配给用户。

根据授权灵活性要求,你可以以不同的方式完成这些任务。
如果只有开发人员更改你的权限层次结构,你可以使用迁移或控制台命令。
迁移的优点是你可以与其他迁移一起执行它。
控制台命令的优点是你可以在代码中很好地概览层次结构,而无需阅读许多迁移。

无论哪种方式,最终你都会得到以下 RBAC 层次结构:

![简单的 RBAC 层次结构](/images/guide/security/rbac-hierarchy-1.svg "简单的 RBAC 层次结构")

如果你想动态构建权限层次结构,则需要 UI 或控制台命令。
用于构建层次结构本身的 API 不会有所不同。

### 使用控制台命令

如果你的权限层次结构根本不会改变,并且你有固定数量的用户,你可以创建一个[控制台命令](../tutorial/console-applications.md),通过 `\Yiisoft\Rbac\ManagerInterface` 提供的 API 一次性初始化授权数据:

```php
<?php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Yiisoft\Rbac\ManagerInterface;
use Yiisoft\Rbac\Permission;
use Yiisoft\Rbac\Role;
use Yiisoft\Yii\Console\ExitCode;

#[AsCommand(
    name: 'rbac:init',
    description: 'Builds RBAC hierarchy',
)]
final readonly class RbacCommand extends Command
{   
    public function __construct(
        private ManagerInterface $manager
    ) {
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $auth = $this->manager;

        $auth->removeAll();                
        
        $createPost = (new Permission('createPost'))->withDescription('创建文章');
        $auth->add($createPost);

        $updatePost = (new Permission('updatePost'))->withDescription('更新文章');
        $auth->add($updatePost);

        // 添加 "author" 角色并赋予此角色 "createPost" 权限
        $author = new Role('author');
        $auth->add($author);
        $auth->addChild($author, $createPost);

        // 添加 "admin" 角色并赋予此角色 "updatePost" 权限
        // 以及 "author" 角色的权限
        $admin = new Role('admin');
        $auth->add($admin);
        $auth->addChild($admin, $updatePost);
        $auth->addChild($admin, $author);

        // 将角色分配给用户。1 和 2 是 IdentityInterface::getId() 返回的 ID
        // 通常在你的 User 模型中实现。
        $auth->assign($author, 2);
        $auth->assign($admin, 1);
        
        return ExitCode::OK;
    }
}
```
 
你可以通过以下方式从控制台执行上述命令:

```
./yii rbac:init
```

> 如果你不想硬编码哪些用户具有某些角色,请不要在命令中放置 `->assign()` 调用。相反,创建 UI 或控制台命令来管理分配。

#### 使用迁移

**TODO**: 在实现迁移时完成它。

你可以使用[迁移](../databases/db-migrations.md)通过 `\Yiisoft\Rbac\ManagerInterface` 提供的 API 初始化和更改层次结构。

使用 `./yii migrate:create init_rbac` 创建新迁移,然后实现创建层次结构:

```php
<?php
use yii\db\Migration;

use Yiisoft\Rbac\ManagerInterface;
use Yiisoft\Rbac\Permission;
use Yiisoft\Rbac\Role;

class m170124_084304_init_rbac extends Migration
{
    public function up()
    {
        $auth = /* obtain auth */;

        $auth->removeAll();                
                
        $createPost = (new Permission('createPost'))->withDescription('创建文章');
        $auth->add($createPost);

        $updatePost = (new Permission('updatePost'))->withDescription('更新文章');
        $auth->add($updatePost);

        // 添加 "author" 角色并赋予此角色 "createPost" 权限
        $author = new Role('author');
        $auth->add($author);
        $auth->addChild($author, $createPost);

        // 添加 "admin" 角色并赋予此角色 "updatePost" 权限
        // 以及 "author" 角色的权限
        $admin = new Role('admin');
        $auth->add($admin);
        $auth->addChild($admin, $updatePost);
        $auth->addChild($admin, $author);

        // 将角色分配给用户。1 和 2 是 IdentityInterface::getId() 返回的 ID
        // 通常在你的 User 模型中实现。
        $auth->assign($author, 2);
        $auth->assign($admin, 1);
    }
    
    public function down()
    {
        $auth = /* obtain auth */;

        $auth->removeAll();
    }
}
```
> 如果你不想硬编码哪些用户具有某些角色,请不要在迁移中放置 `->assign()` 调用。相反,创建 UI 或控制台命令来管理分配。

你可以使用 `./yii migrate` 应用迁移。

## 将角色分配给用户

TODO: 在演示/模板中实现注册时更新。

作者可以创建文章,管理员可以更新文章并执行作者可以执行的所有操作。

如果你的应用程序允许用户注册,你需要立即为这些新用户分配角色。
例如,为了让所有注册用户在你的高级项目模板中成为作者,你需要按如下方式更改 `frontend\models\SignupForm::signup()`:
to change `frontend\models\SignupForm::signup()` as follows:

```php
public function signup()
{
    if ($this->validate()) {
        $user = new User();
        $user->username = $this->username;
        $user->email = $this->email;
        $user->setPassword($this->password);
        $user->generateAuthKey();
        $user->save(false);

        // 添加了以下三行:
        $auth = \Yii::$app->authManager;
        $authorRole = $auth->getRole('author');
        $auth->assign($authorRole, $user->getId());

        return $user;
    }

    return null;
}
```

对于需要具有动态更新授权数据的复杂访问控制的应用程序(例如管理面板),你可能需要使用 `authManager` 提供的 API 开发特殊的用户界面。


### 使用规则 <span id="using-rules"></span>

如前所述,规则为角色和权限添加额外的约束。
规则是从 `\Yiisoft\Rbac\Rule` 扩展的类。
它必须实现 `execute()` 方法。
在你之前创建的层次结构中,作者无法编辑自己的文章。
让我们修复它。首先,你需要一个规则来验证用户是文章作者:

```php
namespace App\User\Rbac;

use Yiisoft\Rbac\Item;
use \Yiisoft\Rbac\Rule;

/**
 * 检查 authorID 是否与通过参数传递的用户匹配。
 */
final readonly class AuthorRule extends Rule
{
    private const NAME = 'isAuthor';

    public function __construct() {
        parent::__construct(self::NAME);
    }

    public function execute(string $userId, Item $item, array $parameters = []): bool
    {
        return isset($params['post']) ? $params['post']->getAuthorId() == $userId : false;
    }
}
```

该规则检查用户是否创建了 `post`。在你之前使用的命令中创建一个特殊权限 `updateOwnPost`:

```php
/** @var \Yiisoft\Rbac\ManagerInterface $auth */

// 添加规则
$rule = new AuthorRule();
$auth->add($rule);

// 添加 "updateOwnPost" 权限并将规则与其关联。
$updateOwnPost = (new \Yiisoft\Rbac\Permission('updateOwnPost'))
    ->withDescription('更新自己的文章')
    ->withRuleName($rule->getName());
$auth->add($updateOwnPost);

// "updateOwnPost" 将从 "updatePost" 使用
$auth->addChild($updateOwnPost, $updatePost);

// 允许 "author" 更新他们自己的文章
$auth->addChild($author, $updateOwnPost);
```

现在你得到了以下层次结构:

![带规则的 RBAC 层次结构](/images/guide/security/rbac-hierarchy-2.svg "带规则的 RBAC 层次结构")


### 访问检查 <span id="access-check"></span>

检查的完成方式与本指南第一部分中的方式类似:

```php
namespace App\Blog\Post;

use Psr\Http\Message\ServerRequestInterface;
use Yiisoft\User\User;

final readonly class PostController
{
    public function actionEdit(ServerRequestInterface $request, User $user, PostRepository $postRepository)
    {
        $postId = $request->getAttribute('id');
        if ($postId === null) {
            // 响应 404
        }
        
        $post = $postRepository->findByPK($postId);
        if ($post === null) {
            // 响应 404
        }

        if (!$this->canEditPost($user, $post)) {
            // 响应 403
        }
        
        // 继续编辑文章
    }
    
    private function canEditPost(User $user, Post $post): bool
    {
        return $user->can('updatePost', ['post' => $post]);    
    }
}
```

不同之处在于,现在检查用户自己的文章是 RBAC 的一部分。

如果当前用户是 `ID=1` 的 Jane,你从 `createPost` 开始并尝试到达 `Jane`:

![访问检查](/images/guide/security/rbac-access-check-1.svg "访问检查")

要检查用户是否可以更新文章,你需要传递之前描述的 `AuthorRule` 所需的额外参数:

```php
if ($user->can('updatePost', ['post' => $post])) {
    // 更新文章
}
```

如果当前用户是 John,会发生以下情况:


![访问检查](/images/guide/security/rbac-access-check-2.svg "访问检查")

你从 `updatePost` 开始并通过 `updateOwnPost`。要通过访问检查,`AuthorRule` 应该从其 `execute()` 方法返回 `true`。该方法从 `can()` 方法调用接收其 `$params`,因此值为 `['post' => $post]`。
如果一切正常,你将到达分配给 John 的 `author`。

对于 Jane 的情况,由于她是管理员,所以更简单一些:

![访问检查](/images/guide/security/rbac-access-check-3.svg "访问检查")

## 实现你自己的访问检查器

如果 RBAC 不适合你的需求,你可以在不更改应用程序代码的情况下实现自己的访问检查器:


```php
namespace App\User;

use \Yiisoft\Access\AccessCheckerInterface;

final readonly class AccessChecker implements AccessCheckerInterface
{
    private const PERMISSIONS = [
        [
            1 => ['editPost'],
            42 => ['editPost', 'deletePost'],
        ],
    ];

    public function userHasPermission($userId, string $permissionName, array $parameters = []) : bool
    {
        if (!array_key_exists($userId, self::PERMISSIONS)) {
            return false;
        }

        return in_array($permissionName, self::PERMISSIONS[$userId], true); 
    }
}
```
