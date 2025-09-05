Z-push的使用方法

以下在AlmaLinux 8配置成功

安装依赖（php-pdo php-mysqlnd php-intl，这三样，缺任何一个都会报错）

系统默认源没有php-imap

```
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm   # Remi 源
sudo dnf module list php
sudo dnf module reset php -y
sudo dnf module enable php:remi-7.2 -y
sudo dnf install -y php-soap php-pdo php-mysqlnd php-intl php-imap
sudo systemctl restart php-fpm
```

```
cd /var/www
sudo wget -4 https://github.com/Z-Hub/Z-Push/archive/refs/tags/2.7.6.tar.gz
sudo tar -xf 2.7.6.tar.gz
sudo mv Z-Push-2.7.6 z-push
```

sudo mkdir -p /var/log/z-push   # z-push默认日志目录，必须手动建，否则出错
sudo mkdir -p /var/lib/z-push
grep ^user /etc/php-fpm.d/www.conf   # 获取php用户，此处是apache，对应修改下面三个命令
sudo chown -R apache:apache /var/www/z-push
sudo chown -R apache:apache /var/log/z-push
sudo chown -R apache:apache /var/lib/z-push

# 修改主配置文件，主要包括日志级别，iphone轮询间隔
sudo wget -4 -O /var/www/z-push/src/config.php https://raw.githubusercontent.com/kswz/z-push/main/src_config.php
# 改动如下（据实修改）：
define('BACKEND_PROVIDER', 'BackendIMAP');
define('TIMEZONE', 'Asia/Shanghai');
define('LOGFILEDIR', '/var/log/z-push/');
define('LOGLEVEL', LOGLEVEL_ERROR);
define('PING_INTERVAL', 30);

# 修改imap配置文件，主要是imap端口，以及各个邮箱子文件夹名称（务必一一对应）
sudo wget -4 -O /var/www/z-push/src/backend/imap/config.php https://raw.githubusercontent.com/kswz/z-push/main/imap_config.php
# 改动如下（据实修改）：
doveadm mailbox list -u user@example.com   # 先列出Dovecot的实际文件夹名称

// 使用本地 Dovecot
define('IMAP_SERVER', 'localhost');
define('IMAP_PORT', 993);
define('IMAP_OPTIONS', '/ssl/novalidate-cert');
define('IMAP_FOLDER_CONFIGURED', true);
# 修改最后一项后面的各个文件夹名称，与前面 doveadm mailbox 完全一致

sudo systemctl restart php-fpm

浏览器访问
https://mail.example.com/Microsoft-Server-ActiveSync
若返回「GET not supported」即表示 PHP 已正常运行。
iOS 端添加 Exchange 账号，服务器填 mail.example.com，用户名用完整的邮件地址，勾选 SSL，测试收发与推送。

*********************************************************************************************
nginx 1.28版反向代理设置：（老版本的http2是写在 443 ssl 的后面，加个http2即可）
```
server {
    listen 443 ssl;
    http2 on;
    server_name mail.example.com;

    root  /var/www/z-push/src;
    index index.php;

    location /Microsoft-Server-ActiveSync {
        try_files $uri /index.php$is_args$args;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        include /etc/nginx/fastcgi.conf;
        fastcgi_intercept_errors on;
    }
}
```

