# 使用数据库

Yii 不强制要求您的应用程序使用特定的数据库或存储。
有多种方式可以使用关系型数据库：

- [Yii DB](https://github.com/yiisoft/db)
- [Yii Active Record](https://github.com/yiisoft/active-record)
- [Cycle](https://github.com/cycle) 通过 [Yii Cycle 包](https://github.com/yiisoft/yii-cycle)
- [Doctrine](https://www.doctrine-project.org/) 通过 [Yii Doctrine 包](https://github.com/stargazer-team/yii-doctrine)
- [PDO](https://www.php.net/manual/en/book.pdo.php)

对于非关系型数据库，通常有官方库可用：

- [ElasticSearch](https://github.com/elastic/elasticsearch-php)
- [Redis](https://redis.io/docs/latest/develop/clients/php/)
- ...

在本指南中，我们将重点介绍使用 Yii DB 处理关系型数据库。我们将使用 PostgreSQL 实现一个简单的 CRUD（创建、读取、更新、删除）功能。

## 安装 PostgreSQL

您需要安装 PostgreSQL。如果您不想使用 Docker，
可以[从官方网站获取安装程序](https://www.postgresql.org/download/)，安装它并创建数据库。

如果您使用 Docker，会更简单一些。修改 `docker/dev/compose.yml`：

```yaml
services:
    app:
        build:
            dockerfile: docker/Dockerfile
            context: ..
            target: dev
            args:
                USER_ID: ${UID}
                GROUP_ID: ${GID}
        env_file:
            -   path: ./dev/.env
            -   path: ./dev/override.env
                required: false
        ports:
            - "${DEV_PORT:-80}:80"
        volumes:
            - ../:/app
            - ../runtime:/app/runtime
            - caddy_data:/data
            - caddy_config:/config
        tty: true
        depends_on:
            db:
                condition: service_healthy

    db:
        image: postgres:${POSTGRES_VERSION:-15}-alpine
        environment:
            POSTGRES_DB: app
            POSTGRES_PASSWORD: password
            POSTGRES_USER: user
        volumes:
            - ../runtime/db:/var/lib/postgresql/data:rw
        ports:
            - "${DEV_DB_PORT:-5432}:5432"
        healthcheck:
            test: [ "CMD-SHELL", "pg_isready -U user -d app" ]
            interval: 5s
            timeout: 5s
            retries: 5

volumes:
    db:
```

注意，我们添加了 `depends_on`，这样应用程序会等待数据库启动。

此外，我们需要 `pdo_pgsql` 扩展来与 PostgreSQL 通信。您可以在本地的 `php.ini` 中启用它。
如果您使用 Docker，请检查 `docker/Dockerfile` 并在 `install-php-extensions` 列表中添加 `pdo_pgsql`。然后使用 `make build && make down && make up` 重新构建 PHP 镜像。

## 配置连接

现在我们有了数据库，是时候定义连接了。

首先我们需要安装一个包：

```sh
make composer require yiisoft/db-pgsql
```

现在创建 `config/common/di/db-pgsql.php`：

```php
<?php

use Yiisoft\Db\Connection\ConnectionInterface;
use Yiisoft\Db\Pgsql\Connection;
use Yiisoft\Db\Pgsql\Driver;

/** @var array $params */

return [
    ConnectionInterface::class => [
        'class' => Connection::class,
        '__construct()' => [
            'driver' => new Driver(
                $params['yiisoft/db-pgsql']['dsn'],
                $params['yiisoft/db-pgsql']['username'],
                $params['yiisoft/db-pgsql']['password'],
            ),
        ],
    ],
];
```

并在 `config/common/params.php` 中定义参数。对于 Docker，应该是：

```php
use Yiisoft\Db\Pgsql\Dsn;

return [
    // ...
    'yiisoft/db-pgsql' => [
        'dsn' => new Dsn('pgsql', 'db', 'app', '5432'),
        'username' => 'user',
        'password' => 'password',
    ],
];
```

`db` 主机在 Docker 网络内自动解析。

对于不使用 Docker 的本地安装，Dsn 中的主机应该是 `localhost`。您需要调整其余部分以匹配您配置数据库的方式。

## 创建和应用迁移

对于应用程序的初始状态和进一步的数据库更改，使用迁移是一个好主意。
这些是创建数据库更改的文件。已应用的迁移会在数据库中跟踪，使我们能够知道当前状态以及还有哪些迁移需要应用。

要使用迁移，我们需要安装另一个包：

```sh
make composer require yiisoft/db-migration
```

在项目根目录中创建一个目录来存储迁移 `src/Migration`。将以下配置添加到 `config/common/params.php`：

```php
'yiisoft/db-migration' => [
    'newMigrationNamespace' => 'App\\Migration',
    'sourceNamespaces' => ['App\\Migration'],
],
```

现在您可以使用 `make yii migrate:create page` 创建新的迁移。对于我们的示例，我们需要一个带有一些列的 `page` 表：

```php
<?php

declare(strict_types=1);

namespace App\Migration;

use Yiisoft\Db\Migration\MigrationBuilder;
use Yiisoft\Db\Migration\RevertibleMigrationInterface;

final class M251102141707Page implements RevertibleMigrationInterface
{
    public function up(MigrationBuilder $b): void
    {
        $cb = $b->columnBuilder();

        $b->createTable('page', [
            'id' => $cb::uuidPrimaryKey(),
            'title' => $cb::string()->notNull(),
            'slug' => $cb::string()->notNull()->unique(),
            'text' => $cb::text()->notNull(),
            'created_at' => $cb::dateTime(),
            'updated_at' => $cb::dateTime(),
        ]);
    }

    public function down(MigrationBuilder $b): void
    {
        $b->dropTable('page');
    }
}
```

注意，我们使用 UUID 作为主键。我们将自己生成这些 ID，而不是依赖数据库，因此我们需要一个额外的 composer 包。

```shell
make composer require ramsey/uuid
```

虽然存储空间比使用 int 稍大，但使用这种 ID 的工作流程是有益的。由于您自己生成 ID，因此可以定义一组相关数据并在单个事务中保存它。
在代码中定义这组数据的实体通常称为"聚合"。

使用 `make yii migrate:up` 应用迁移。

## 实体

现在您有了一个表，是时候在代码中定义一个实体了。创建 `src/Web/Page/Page.php`：

```php
<?php

declare(strict_types=1);

namespace App\Web\Page;

use DateTimeImmutable;
use Yiisoft\Strings\Inflector;

final readonly class Page
{
    private function __construct(
        public string $id,
        public string $title,
        public string $text,
        public DateTimeImmutable $createdAt,
        public DateTimeImmutable $updatedAt,
    ) {}

    public static function create(
        string $id,
        string $title,
        string $text,
        ?DateTimeImmutable $createdAt = null,
        ?DateTimeImmutable $updatedAt = null,
    ): self {
        return new self(
            id: $id,
            title: $title,
            text: $text,
            createdAt: $createdAt ?? new DateTimeImmutable(),
            updatedAt: $updatedAt ?? new DateTimeImmutable(),
        );
    }

    public function getSlug(): string
    {
        return (new Inflector())->toSlug($this->title);
    }
}
```

## 仓储

现在我们有了实体，我们需要一个地方来放置保存实体、删除实体以及选择单个页面或多个页面的方法。

创建 `src/Web/Page/PageRepository.php`：

```php
<?php

declare(strict_types=1);

namespace App\Web\Page;

use DateTimeImmutable;
use Yiisoft\Db\Connection\ConnectionInterface;
use Yiisoft\Db\Query\Query;

final readonly class PageRepository
{
    public function __construct(
        private ConnectionInterface $connection,
    ) {}

    public function save(Page $page): void
    {
        $data = [
            'id' => $page->id,
            'title' => $page->title,
            'slug' => $page->getSlug(),
            'text' => $page->text,
            'created_at' => $page->createdAt,
            'updated_at' => $page->updatedAt,
        ];

        if ($this->exists($page->id)) {
            $this->connection->createCommand()->update('{{%page}}', $data, ['id' => $page->id])->execute();
        } else {
            $this->connection->createCommand()->insert('{{%page}}', $data)->execute();
        }
    }

    public function findOneBySlug(string $slug): ?Page
    {
        $query = $this->connection
            ->select()
            ->from('{{%page}}')
            ->where('slug = :slug', ['slug' => $slug]);

        return $this->createPage($query->one());
    }

    /**
     * @return iterable<Page>
     */
    public function findAll(): iterable
    {
        $data = $this->connection
            ->select()
            ->from('{{%page}}')
            ->all();

        foreach ($data as $page) {
            yield $this->createPage($page);
        }
    }

    private function createPage(?array $data): ?Page
    {
        if ($data === null) {
            return null;
        }

        return Page::create(
            id: $data['id'],
            title: $data['title'],
            text: $data['text'],
            createdAt: new DateTimeImmutable($data['created_at']),
            updatedAt: new DateTimeImmutable($data['updated_at']),
        );
    }

    public function deleteBySlug(string $slug): void
    {
        $this->connection->createCommand()->delete(
            '{{%page}}',
            ['slug' => $slug],
        )->execute();
    }

    public function exists(string $id): bool
    {
        return $this->connection->createQuery()
            ->from('{{%page}}')
            ->where(['id' => $id])
            ->exists();
    }
}
```

在这个仓储中，既有获取数据的方法，也有执行插入或更新的 `save()` 方法。数据库返回原始数据作为数组，但我们的仓储会自动从这些原始数据创建实体，因此稍后我们操作的是类型化数据。

## 操作和路由

我们需要一些操作来：

1. 列出所有页面。
2. 查看一个页面。
3. 删除一个页面。
4. 创建一个页面。
5. 更新一个页面。

然后我们需要为所有这些设置路由。

让我们逐一解决这些问题。

### 列出所有页面

创建 `src/Web/Page/ListAction.php`：

```php
<?php

declare(strict_types=1);

namespace App\Web\Page;

use Psr\Http\Message\ResponseInterface;
use Yiisoft\Yii\View\Renderer\ViewRenderer;

final readonly class ListAction
{
    public function __construct(
        private ViewRenderer $viewRenderer,
        private PageRepository $pageRepository,
    )
    {
    }

    public function __invoke(): ResponseInterface
    {
        return $this->viewRenderer->render(__DIR__ . '/list', [
            'pages' => $this->pageRepository->findAll(),
        ]);
    }
}
```

在 `src/Web/Page/list.php` 中定义列表视图：

```php
<?php
use App\Web\Page\Page;
use Yiisoft\Html\Html;
use Yiisoft\Router\UrlGeneratorInterface;

/** @var iterable<Page> $pages */
/** @var UrlGeneratorInterface $urlGenerator */
?>

<ul>
    <?php foreach ($pages as $page): ?>
    <li>
        <?= Html::a($page->title, $urlGenerator->generate('page/view', ['slug' => $page->getSlug()])) ?>
    </li>
    <?php endforeach ?>
</ul>

<?= Html::a('Create', $urlGenerator->generate('page/edit', ['slug' => 'new'])) ?>
```

### 查看一个页面

创建 `src/Web/Page/ViewAction.php`：

```php
<?php

declare(strict_types=1);

namespace App\Web\Page;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Yiisoft\Http\Status;
use Yiisoft\Router\HydratorAttribute\RouteArgument;
use Yiisoft\Yii\View\Renderer\ViewRenderer;

final readonly class ViewAction
{
    public function __construct(
        private ViewRenderer $viewRenderer,
        private PageRepository $pageRepository,
        private ResponseFactoryInterface $responseFactory,
    ) {}

    public function __invoke(
        #[RouteArgument('slug')]
        string $slug,
    ): ResponseInterface {
        $page = $this->pageRepository->findOneBySlug($slug);
        if ($page === null) {
            return $this->responseFactory->createResponse(Status::NOT_FOUND);
        }

        return $this->viewRenderer->render(__DIR__ . '/view', [
            'page' => $page,
        ]);
    }
}
```

现在，在 `src/Web/Page/view.php` 中创建模板：

```php
<?php
use App\Web\Page\Page;
use Yiisoft\Html\Html;
use Yiisoft\Router\UrlGeneratorInterface;
use Yiisoft\Yii\View\Renderer\Csrf;

/** @var Page $page */
/** @var UrlGeneratorInterface $urlGenerator */
/* @var Csrf $csrf */
?>

<h1><?= Html::a('Pages', $urlGenerator->generate('page/list')) ?> → <?= Html::encode($page->title) ?></h1>

<p>
    <?= Html::encode($page->text) ?>
</p>

<?= Html::a('Edit', $urlGenerator->generate('page/edit', ['slug' => $page->getSlug()])) ?> |


<?php
    $deleteForm = Html::form()
        ->post($urlGenerator->generate('page/delete', ['slug' => $page->getSlug()]))
        ->csrf($csrf);
?>
<?= $deleteForm->open() ?>
    <?= Html::submitButton('Delete') ?>
<?= $deleteForm->close() ?>
```

在这个视图中，我们有一个提交页面删除请求的表单。使用 `GET` 处理它也很常见，但这是非常错误的。由于删除会更改数据，因此需要由非幂等的 HTTP 方法之一来处理。
我们在示例中使用 POST 和表单，但也可以是 `DELETE` 和使用 JavaScript 发出的异步请求。
该按钮稍后可以适当地设置样式，使其看起来类似于"编辑"。

### 删除一个页面

创建 `src/Web/Page/DeleteAction.php`：

```php
<?php

declare(strict_types=1);

namespace App\Web\Page;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Yiisoft\Http\Status;
use Yiisoft\Router\HydratorAttribute\RouteArgument;
use Yiisoft\Router\UrlGeneratorInterface;

final readonly class DeleteAction
{
    public function __construct(
        private PageRepository $pageRepository,
        private ResponseFactoryInterface $responseFactory,
        private UrlGeneratorInterface $urlGenerator,
    ) {}

    public function __invoke(
        #[RouteArgument('slug')]
        string $slug
    ): ResponseInterface
    {
        $this->pageRepository->deleteBySlug($slug);

        return $this->responseFactory
            ->createResponse(Status::SEE_OTHER)
            ->withHeader('Location', $this->urlGenerator->generate('page/list'));
    }
}
```

### 创建或更新一个页面

创建 `src/Web/Page/EditAction.php`：

```php
<?php

declare(strict_types=1);

namespace App\Web\Page;

use DateTimeImmutable;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Ramsey\Uuid\Uuid;
use Yiisoft\FormModel\FormHydrator;
use Yiisoft\Http\Status;
use Yiisoft\Router\HydratorAttribute\RouteArgument;
use Yiisoft\Router\UrlGeneratorInterface;
use Yiisoft\Yii\View\Renderer\ViewRenderer;

final readonly class EditAction
{
    public function __construct(
        private ViewRenderer $viewRenderer,
        private FormHydrator $formHydrator,
        private ResponseFactoryInterface $responseFactory,
        private UrlGeneratorInterface $urlGenerator,
    ) {}

    public function __invoke(
        #[RouteArgument('slug')]
        string $slug,
        ServerRequestInterface $request,
        PageRepository $pageRepository,
    ): ResponseInterface
    {
        $isNew = $slug === 'new';

        $form = new Form();

        if (!$isNew) {
            $page = $pageRepository->findOneBySlug($slug);
            if ($page === null) {
                return $this->responseFactory->createResponse(Status::NOT_FOUND);
            }

            $form->title = $page->title;
            $form->text = $page->text;
        }

        $this->formHydrator->populateFromPostAndValidate($form, $request);

        if ($form->isValid()) {
            $id = $isNew ? Uuid::uuid7()->toString() : $page->id;

            $page = Page::create(
                id: $id,
                title: $form->title,
                text: $form->text,
                updatedAt: new DateTimeImmutable(),
            );

            $pageRepository->save($page);

            return $this->responseFactory
                ->createResponse(Status::SEE_OTHER)
                ->withHeader(
                    'Location',
                    $this->urlGenerator->generate('page/view', ['slug' => $page->getSlug()]),
                );
        }

        return $this->viewRenderer->render(__DIR__ . '/edit', [
            'form' => $form,
            'isNew' => $isNew,
            'slug' => $slug,
        ]);
    }
}
```

在上面的代码中，我们在 URL 中为新页面使用了一个特殊的 slug，因此 URL 看起来像 `http://localhost/pages/new`。如果页面不是新的，我们会用数据库中的数据预填充表单。与我们在[使用表单](forms.md)中所做的类似，我们处理表单提交。成功保存后，我们重定向到页面视图。

现在，在 `src/Web/Page/edit.php` 中创建模板：

```php
<?php
use App\Web\Page\Form;
use Yiisoft\FormModel\Field;
use Yiisoft\Html\Html;
use Yiisoft\Router\UrlGeneratorInterface;
use Yiisoft\Yii\View\Renderer\Csrf;

/**
 * @var Form $form
 * @var string[] $errors
 * @var UrlGeneratorInterface $urlGenerator
 * @var Csrf $csrf
 * @var bool $isNew
 * @var string $slug
 */

$htmlForm = Html::form()
    ->post($urlGenerator->generate('page/edit', ['slug' => $slug]))
    ->csrf($csrf);
?>

<?= $htmlForm->open() ?>
    <?= Field::text($form, 'title')->required() ?>
    <?= Field::textarea($form, 'text')->required() ?>
    <?= Html::submitButton('Save') ?>
<?= $htmlForm->close() ?>
```

### 路由

调整 `config/common/routes.php`：

```php
<?php

declare(strict_types=1);

use App\Web;
use Yiisoft\Http\Method;
use Yiisoft\Router\Group;
use Yiisoft\Router\Route;

return [
    Group::create()
        ->routes(
            Route::get('/')
                ->action(Web\HomePage\Action::class)
                ->name('home'),
            Route::methods([Method::GET, Method::POST], '/say')
                ->action(Web\Echo\Action::class)
                ->name('echo/say'),

            Group::create('/pages')->routes(
                Route::get('')
                    ->action(Web\Page\ListAction::class)
                    ->name('page/list'),
                Route::get('/{slug}')
                    ->action(Web\Page\ViewAction::class)
                    ->name('page/view'),
                Route::methods([Method::GET, Method::POST], '/{slug}/edit')
                    ->action(Web\Page\EditAction::class)
                    ->name('page/edit'),
                Route::post('/{slug}/delete')
                    ->action(Web\Page\DeleteAction::class)
                    ->name('page/delete'),
            ),
        ),
];
```

注意，我们已经将所有与页面相关的路由分组到 `/pages` 前缀下。这是一种方便的方式，既可以避免重复，又可以向整个组添加一些额外的中间件，例如身份验证。

## 试用

现在在浏览器中打开 `http://localhost/pages` 试用一下。
