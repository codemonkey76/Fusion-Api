##Instructions

###Login to fusionpbx server as root and clone this repo into /var/www/fusionapi, and give www-data user ownership
```
sudo su
git clone https://github.com/codemonkey76/Fusion-Api.git /var/www/fusionapi
chown -R www-data:www-data /var/www/fusionapi
```
###Install dependencies
```
apt-get install mariadb-server
```

###Install composer
```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === '906a84df04cea2aa72f40b5f787e49f22d4c2f19492ac310e8cba5b96ac8b64115ac402c8cd292b8a03482574915d1a8') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php --install-dir=/usr/bin --filename=composer
php -r "unlink('composer-setup.php');"
```

###Install composer dependencies and generate app key
```
su -l www-data -s /bin/bash
cd /var/www/fusionapi
cp .env.example .env
composer install
php artisan key:generate
```

###Setup database
```
mysql
create database fusionapi;
create user 'fusionapi'@'localhost' identified by 'secret';
grant all privilieges on fusionapi.* to 'fusionapi'@'localhost';
flush privileges;
exit
```

###Setup .env file using your favorite editor set the following fields
```
DB_DATABASE=fusionapi
DB_USERNAME=fusionapi
DB_PASSWORD=secret
...
DB2_DATABASE=fusionpbx
DB2_USERNAME=fusionpbx
DB2_PASSWORD=fusionpbx
```

###Set fusionpbx password
sed -i "s/DB2_PASSWORD.*$/DB2_PASSWORD=$(cat /etc/fusionpbx/config.php | grep '$db_password
 = ' | awk -F\' '{print $2}')/g" .env
```

##Step 1. Create nginx config file
Step 1. Create nginx config file in /etc/nginx/sites-available/fusionapi
```
server {
        listen 81;
        listen [::]:81;


        root /var/www/fusionapi;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name s02.alphapbx.com.au;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.3-fpm.sock;
        }

        location ~ /\.ht {
                deny all;
        }
}
```

Step 2. Add to sites-enabled
ln -s /etc/nginx/sites-available/fusionapi /etc/nginx/sites-enabled/fusionapi

Step 3. Restart nginx
service nginx restart

Step 4. Allow port 81 through IP Tables
sudo iptables -A INPUT -p tcp --dport 81 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 81 -m conntrack --ctstate ESTABLISHED -j ACCEPT

Step 5. Save rules
dpkg-reconfigure iptables-persistent
"Choose yes to save rules"

Step 6. Create index.php in /var/www/fusionapi containing
<?php
phpinfo();
?>

Step 7. Set permissions
chown www-data www-data -R /var/www/fusionapi

Step 8. Visit port 81 ensure that php info page is displayed