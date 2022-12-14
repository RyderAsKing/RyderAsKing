# WHMCS installation on a slice
WHMCS installation can be rough sometimes
## Update and upgrade your packages:

sudo apt update
sudo apt upgrade
## Install Nginx and start it:

sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
## Install, start and set up MariaDB:

sudo apt install mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
## Install dependencies:

sudo apt install ca-certificates apt-transport-https software-properties-common wget curl lsb-release
## Import a repository for PHP 8.1 and then update and upgrade packages again:

sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update

## Install PHP 8.1 and then start FPM and check its running:

sudo apt install php8.1 php8.1-fpm php8.1-cli
sudo systemctl start php8.1-fpm
sudo systemctl enable php8.1-fpm
sudo systemctl status php8.1-fpm
## Install the extensions required by WHMCS:

sudo apt install php8.1-curl php8.1-gd php8.1-xml php8.1-common php8.1-mbstring php8.1-gmp php8.1-bcmath php8.1-intl php8.1-zip php8.1-mysql php8.1-soap
## Download and extract ioncube loaders:

cd /tmp
wget https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz
tar -zxvf ioncube_loaders_lin_x86-64.tar.gz
## Get your PHP extension path:

php -i | grep extension_dir
>> extension_dir => /usr/lib/php/20210902 => /usr/lib/php/20210902
## Copy the correct loader to your extensions directory:

sudo cp /tmp/ioncube/ioncube_loader_lin_8.1.so /usr/lib/php/20210902
## Add the extension to your php.ini file (you may also want to add it to /etc/php/8.1/cli/php.ini)  :

sudo nano /etc/php/8.1/fpm/php.ini
zend_extension = /usr/lib/php/20210902/ioncube_loader_lin_8.1.so
## Restart PHP FPM:

sudo systemctl restart php8.1-fpm
 

## Set up your Nginx configuration file (in my example debian.leemahoney.cloud)  :

sudo nano /etc/nginx/sites-available/debian.leemahoney.cloud
```
server {
        listen 80;
        server_name debian.leemahoney.cloud;

        root /var/www/html;
        access_log /var/log/nginx/debian.leemahoney.cloud-access_log;
        error_log /var/log/nginx/debian.leemahoney.cloud-error_log;

        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";

        index index.php index.html index.htm;

        charset utf-8;

        location / {
                try_files $uri $uri/ /index.php?$query_string;
        }

        location = /favicon.ico { access_log off; log_not_found off; }
        location = /robots.txt  { access_log off; log_not_found off; }

        error_page 404 /index.php;


        proxy_send_timeout 300s;
        proxy_read_timeout 300s;


        location ~ \.php$ {
                fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
                fastcgi_send_timeout 300;
                fastcgi_read_timeout 300;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
                include fastcgi_params;
        }

		location ~ /announcements/?(.*)$ {
                rewrite ^/(.*)$ /index.php?rp=/announcements/$1;
        }

        location ~ /downloads/?(.*)$ {
                rewrite ^/(.*)$ /index.php?rp=/downloads/$1;
        }

        location ~ /knowledgebase/?(.*)$ {
                rewrite ^/(.*)$ /index.php?rp=/knowledgebase/$1;
        }

        location ~ /\.(?!well-known).* {
                deny all;
        }


        location ^~ /vendor/ {
                deny all;
                return 403;
        }

        location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc|svg|woff|woff2|ttf)\$ {
                expires 1M;
                access_log off;
                add_header Cache-Control "public";
        }

        location ~* \.(?:css|js)\$ {
                expires 7d;
                access_log off;
                add_header Cache-Control "public";
        }

        location ~ /\.ht {
                deny  all;
        }
}
```
Modified from https://gist.github.com/Bharat-B/6ba2e18f85591c77fdf00ad7334fb9c6 and https://gist.github.com/Bharat-B/62205bfd1dbe6d7ac9e24973c2bfd47e

## Enable the configuration file and reload Nginx:

sudo ln -s /etc/nginx/sites-available/debian.leemahoney.cloud /etc/nginx/sites-enabled/debian.leemahoney.cloud
sudo systemctl reload nginx
## Optionally install a free SSL on the domain:

sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d debian.leemahoney.cloud
## Create a database and a user for WHMCS:

sudo mysql -u root -p

>> create database whmcs;
>> create user whmcsuser@localhost identified by 'mystrongpassword';
>> grant delete, insert, select, update, lock tables, alter, create, drop, index on whmcs.* to whmcsuser@localhost;
>> flush privileges;
>> exit
## Download WHMCS v8.6 to your /var/www/html folder and enjoy. (will not work on versions prior due to PHP 8.1)

## Ps, don't forget to change permissions on your html folder:

sudo chown -R $USER:$USER /var/www/html
