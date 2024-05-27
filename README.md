# linkstack-setup-lnmp
this setup is for deploying linkstack service on self-hosting server with LNMP environment

before any proceeding please make sure the environment is installed

1 install php extension for linkstack to work

```shell
sudo apt install php-fpm php-bcmath php-ctype php-curl php-dom php-fileinfo php-json php-mbstring php-pdo php-tokenizer php-iconv php-xml php-mysql php-gd php-cli php-zip -y
```

2 next, create a database for linkstack using MariaDB

3 go to the directory of nginx root dir, and download the linkstack zip file 

```shell
cd /var/www/html
wget https://github.com/linkstackorg/linkstack/releases/latest/download/linkstack.zip && unzip linkstack.zip && rm linkstack.zip 
```

and then change the permission of this file, so the nginx server can access and modify it 

```shell
sudo chown -R www-data:www-data linkstack && sudo chmod -R 755 linkstack
```

4 set up your domain name record, and upload your SSL certificate to the server, let's say your put your certificate under `/var/www/cert/` ,that's what I normally do so can manage the certificate eaisly

5 go to `/etc/nginx/sites-available` to create a linkstack configuration file named `linkstack.conf` and put the content into it and modify to suit your actual case 

```nginx
server {
    server_name  _;
    return 302 $scheme://example.com$request_uri;
}

server {
    listen 80;
    server_name example.com;
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name example.com;
    
    root /var/www/html/linkstack/;
    index index.php;
    
    ssl_certificate /etc/letsencrypt/live/example.com.pem;
    ssl_certificate_key /etc/letsencrypt/live/.example.com.pem;
    
    ssl_session_cache shared:SSL:20m;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5';

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
    add_header Referrer-Policy origin always; # make sure outgoing links don't show the URL to the instance
    add_header X-XSS-Protection "1; mode=block" always;

    charset utf-8;
    
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        # With php-fpm (or other unix sockets):
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        # With php-cgi (or other tcp sockets):
        # fastcgi_pass 127.0.0.1:9000;
        client_max_body_size 20m;
        fastcgi_connect_timeout 30s;
        fastcgi_send_timeout 30s;
        fastcgi_read_timeout 30s;
        fastcgi_intercept_errors on;
	}

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ /\.(?!well-known).* {
        deny all;
    }


    location ~ ^\. {
    deny all;
    }

    location ~ \.sqlite$ {
    deny all;
    }

    location ~ \.env$ {
    deny all;
    }

    location ~ /\.htaccess {
    allow all;
    }

}
```

and then create a symbol link to `/etc/nginx/sites-enabled/`

```shell
cd /etc/nginx/sites-enabled
sudo ln -s /etc/nginx/sites-available/linkstack.conf
```



6 since linkstack is created by Laravel framework, to make sure this whole Laravel project can work, all the extensions are listed in the `composer.json` file in the linkstack directory, we need `composer install` to check if al the extensions are set up, but before that we need to install `composer` to run `composer install` 

firstly, install the `composer` in the bin directory so it can be called in the global environment 

```shell
cd /usr/local/bin
curl -sS https://getcomposer.org/installer | php
```

and then rename it as `composer` 

```shell
mv composer.phar composer
```

and then go to `/var/www/html/linstack/` where the `composer.json` file is in the pwd , and then run the commands, need to check out why suggested not run this command as root user 

```shell
composer update
composer install
```

7 go to your linkstack file to edit the `.env` file which is the configuration file for Laravel project , you can use `ls -a` to check it out , make sure all the information below are correctly setup, find this line 

```php
#Database Settings=Should be left alone. If you wish to use mysql you'd have to seed the database again.
DB_CONNECTION=sqlite
```

and then replace it with the content below, change your database info only, i.e. name, account and password

```php
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=linkstack
DB_USERNAME=linkstack-user
DB_PASSWORD=linkstack-password
```

after update the database information in the `.env` we still need to migrate our database , run this command 

```shell
php artisan migrate
```



8 finally restart nginx and php-fpm, it should works 

```shell
nginx -t && sudo systemctl restart nginx && sudo systemctl restart php8.2-fpm
```








