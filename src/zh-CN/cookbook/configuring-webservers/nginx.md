# 配置 Web 服务器：Nginx

要使用 [Nginx](https://wiki.nginx.org/)，请将 PHP 安装为 [FPM SAPI](https://secure.php.net/install.fpm)。
使用以下 Nginx 配置，将 `path/to/app/public` 替换为 `app/public` 的实际路径，将 `mysite.test` 替换为要提供服务的实际主机名。

```nginx
server {
    charset utf-8;
    client_max_body_size 128M;

    listen 80; ## listen for ipv4
    #listen [::]:80 default_server ipv6only=on; ## listen for ipv6

    server_name mysite.test;
    root        /path/to/app/public;
    index       index.php;

    access_log  /path/to/basic/log/access.log;
    error_log   /path/to/basic/log/error.log;

    location / {
        # Redirect everything that isn't a real file to index.php
        try_files $uri $uri/ /index.php$is_args$args;
    }

    # uncomment to avoid processing of calls to non-existing static files by Yii
    #location ~ \.(js|css|png|jpg|gif|swf|ico|pdf|mov|fla|zip|rar)$ {
    #    try_files $uri =404;
    #}
    #error_page 404 /404.html;

    # deny accessing php files for the /assets directory
    location ~ ^/assets/.*\.php$ {
        deny all;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass 127.0.0.1:9000;
        #fastcgi_pass unix:/var/run/php8-fpm.sock;
        try_files $uri =404;
        fastcgi_param APP_ENV "dev";
    }

    location ~* /\. {
        deny all;
    }
}
```

使用此配置时，还要在 `php.ini` 文件中设置 `cgi.fix_pathinfo=0`，以避免许多不必要的系统 `stat()` 调用。

另外，请注意，在运行 HTTPS 服务器时，您需要添加 `fastcgi_param HTTPS on;`，以便 Yii 可以检测连接是否安全。

在上面的配置中，请注意 `fastcgi_param APP_ENV` 的使用。由于 Yii3 应用程序模板使用环境变量，这是设置它们的一个可能位置。在生产环境中，请记住将 `APP_ENV` 设置为 `prod`。
