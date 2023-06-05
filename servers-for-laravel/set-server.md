# SET SERVER FOR LARAVEL

## Installing php and nginx

### Installing the Software

```shell
sudo apt update
```

```shell
sudo apt install nginx php8.2 php8.2-fpm
```

```shell
sudo apt install zip unzip php8.2-zip php8.2-curl php8.2-xml
```

```shell
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
```

```shell
php composer-setup.php
```

```shell
sudo mv composer.phar /usr/local/bin/composer
```

### Verifying the Installation

```shell
composer version
```

```shell
nginx -v
```

### Allowing Access on Port 80 and 443

To allow access on port 80 and 443, we need to add a new rule to our stack template file and
deploy it using `aws cloud formation deploy`.

## Installing the Laravel App

### Generating an SSH Key and Adding it to the GitHub Repo

```shell
ssh-keygen -f ~/.ssh/id_rsa -t rsa -P ""
```

```shell
cat ~/.ssh/id_rsa.pub
```

* Copy the public key and add it as a deployment key on GitHub by going to the repo's settings tab
  and selecting "Deploy Keys."

### Cloning the Repo and Installing Dependencies

```shell
cd /var/www
git clone <repo information>
composer install
```

### Making www-data the Owner of the www Directory

```shell
sudo chown www-data:www-data /var/www
```

## Configuring Nginx

```shell
cd /etc/nginx/sites-available
touch <laravel>
sudo nano <laravel>
```

Paste configuration of laravel to `<laravel>` config file

```shell
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    root /var/www/laracast/public;

    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
    }
}
```

```shell
sudo ln -s /etc/nginx/sites-available/laracast /etc/nginx/sites-enabled/
```

* Delete default config from `sites-available` and `sites-enabled`

```shell
sudo service nginx reload
```

## Install and Secure MySQL 8

```shell
sudo apt install mysql-server -y
```

```shell
sudo mysql
```

```mysql
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '12345678';
```

```shell
sudo mysql_secure_installation
```

```shell
mysql -u root -p
```

```mysql
CREATE DATABASE laracasts CHARACTER SET utf8 COLLATE utf8_unicode_ci;
```

```mysql
CREATE USER 'laracasts'@'localhost' IDENTIFIED BY '12345678';
```

```mysql
GRANT ALL PRIVILEGES ON laracasts.* TO 'laracasts'@'localhost';
```

```mysql
FLUSH PRIVILEGES;
```

```shell
sudo apt install php8.2-mysql
```

## Installing Redis and Running Workers

```shell
sudo apt install redis-server -y
```

```shell
sudo apt install php8.2-redis
```

```shell
sudo apt install supervisor
```

```shell
cd /etc/supervisor/conf.d
sudo nano laracasts-workers.conf
```

```text
[program:laracasts-workers]
process_name=%(program_name)s_%(process_num)02d
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
redirect_stderr=true
stopwaitsecs=3600
command=php8.2 /home/laracasts/www/artisan queue:work
user=laracasts
stdout_logfile=/home/laracasts/www/storage/logs/workers.log
numprocs=3
```

```shell
touch /home/laracasts/www/storage/logs/workers.log
```

```shell
sudo chown laracasts:laracasts /home/laracasts/www/storage/logs/workers.log
```

```shell
sudo supervisorctl reread
```

```shell
sudo supervisorctl update
```

```shell
sudo supervisorctl status laracasts-worker:*
```
