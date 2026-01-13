# 配置 Web 服务器：Apache

在 Apache 的 `httpd.conf` 文件或虚拟主机配置中使用以下配置。请注意，您应该将 `path/to/app/public` 替换为 `app/public` 的实际路径。

```apache
# Set document root to be "app/public"
DocumentRoot "path/to/app/public"

<Directory "path/to/app/public">
    # use mod_rewrite for pretty URL support
    RewriteEngine on
    
    # if $showScriptName is false in UrlManager, do not allow accessing URLs with script name
    RewriteRule ^index.php/ - [L,R=404]
    
    # If a directory or a file exists, use the request directly
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    
    # Otherwise forward the request to index.php
    RewriteRule . index.php
    
    SetEnv APP_ENV dev

    # ...other settings...
</Directory>
```

如果您有 `AllowOverride All`，您可以添加 `.htaccess` 文件并使用以下配置，而不是使用 `httpd.conf`：

```apache
# use mod_rewrite for pretty URL support
RewriteEngine on

# if $showScriptName is false in UrlManager, do not allow accessing URLs with script name
RewriteRule ^index.php/ - [L,R=404]

# If a directory or a file exists, use the request directly
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d

# Otherwise forward the request to index.php
RewriteRule . index.php

SetEnv APP_ENV dev

# ...other settings...
```

在上面的配置中，请注意 `SetEnv` 的使用。由于 Yii3 应用程序模板使用环境变量，这是设置它们的一个可能位置。在生产环境中，请记住将 `APP_ENV` 设置为 `prod`。
