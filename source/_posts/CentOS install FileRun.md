---


---



# Install FileRun



## Install Nginx Via Yum

install nginx via yum

1. add sources

```bash
rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
```

2. install nginx

```bash
yum -y install nginx
```

3. verify and start

```bash
nginx -v
```

 if successed ,you will see the version of nginx

```
systemctl start nginx
```





## Install MySQL

1. add sources

```bash
cd 
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
```

2. install mysql sources and mysql

```bash
yum -y install mysql57-community-release-el7-10.noarch.rpm
yum -y install mysql-community-server
```

### MySQL Setup

1. start mysql

```bash
systemctl start mysql
```

2. get temp password

```bash
grep "password" /var/log/mysqld.log
```

3. login and modify password

```mysql
mysql -uroot -p
> ******
# you can't do anything before you modify the password
alter user 'root'@'localhost' identified by 'your_new_password';
# your password must be strong enough
# or you can do this
set global validate_password_policy=0;
set global validate_password_length=1;
# and modify a new simple password
#...

```

4. then uninstall sources

```
yum -y remove mysql57-community-release-el7-10.noarch
```

5. modify character-set to utf8

```bash
vim /etc/my.cnf
```

add those 

```ini
[client]
default-character-set=utf8

[mysqld]
character-set-server=utf8

[mysql]
default-character-set=utf8

```

restart mysqld

```bash
systemctl restart mysqld
```

notice:if you get any error you can see mysqld.log like this

```bash
grep "ERROR" /var/log/mysqld.log
```

then verify if the settings is validte

```mysql
mysql -uroot -p
> ******
show variables like 'character_set%';
# you can see result
```

[Reference install mysql](https://blog.csdn.net/baidu_32872293/article/details/80557668)

### Create FileRun Database and Account

```mysql
mysql -uroot -p
> ******
# create database
create database filerun;
# create user
create user 'filerun'@'localhost' identified by 'your_password';
# grant privileges
grant all privileges to filerun.* to 'filerun'@'localhost' identified by 'your_password';
flush privileges;

```

[Reference filerun database settings](https://timelate.com/archives/install-filerun-on-ubuntu.html)

## Install PHP7.1 via yum

1. add sources

```bash
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
```

2. install php and verify it

```bash
yum install php71w php71w-fpm php71w-cli php71w-common php71w-devel php71w-gd php71w-pdo php71w-mysql php71w-mbstring php71w-bcmath
# after those
php -v
```

if you can't install pdo_mysql module , you can see this [Linux安装php-mysql提示需要：libmysqlclient.so.18()(64bit)的解决办法](https://blog.csdn.net/ckg8933/article/details/83379279)

3. install ioncube for php

```bash
cd 
# for 32bit system
wget http://downloads2.ioncube.com/loader_downloads/ioncube_loaders_lin_x86.tar.gz

# for 64bit system
wget http://downloads2.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz


tar xvfz ioncube_loaders_lin_x86-64.tar.gz
# copy to your php lib (selectable)
# lookup php lib
php -i | grep modules
extension_dir => /usr/lib64/php/modules => /usr/lib64/php/modules

cp ioncube/ioncube_loader_lin_7.1.so /usr/lib64/php/modules

# then edit php config to add extend

vim /etc/php.d/php.ini

# restart php-fpm and verify
systemctl restart php-fpm

php -v
PHP 7.1.31 (cli) (built: Aug  4 2019 09:25:59) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.1.0, Copyright (c) 1998-2018 Zend Technologies
    with the ionCube PHP Loader + ionCube24 v10.3.7, Copyright (c) 2002-2019, by ionCube Ltd.
```

[Reference](http://www.thinkphp.cn/topic/50696.html)

## Install FileRun

1. install filerun

```bash
cd /usr/share/nginx/html
mkdir filerun && cd filerun

wget -c "http://www.filerun.com/download-latest" -O filerun.zip && unzip filerun.zip && rm filerun.zip

cd ..
chown -R apache:apache filerun

```

2. nginx config 

   ```nginx
   # rewrite 80 to 443
   server {
   	listen 80;
   	server_name example.com;
   	rewrite ^(.*)$ https://$host$1 permanent;
   }
   
   server {
       listen 443 ssl; # listen 80; if no ssl
       index index.php;   
       server_name  example.com;
       root /usr/share/nginx/html/filerun;
   
   	# ssl_certificate /etc/nginx/cert/yourca.pem;	
   	# ssl_certificate_key /etc/nginx/cert/yourkey.key;
   	add_header Strict-Transport-Security "max-age=15768000;includeSubDomains; preload;";
       add_header X-Content-Type-Options nosniff;
       add_header X-Frame-Options "SAMEORIGIN";
       add_header X-XSS-Protection "1; mode=block";
       add_header X-Robots-Tag none;
       add_header X-Download-Options noopen;
       add_header X-Permitted-Cross-Domain-Policies none;	
   
       # individual nginx logs for this gitlab vhost
       access_log  /var/log/nginx/cloud.zzserver.top_access.log;
       error_log   /var/log/nginx/cloud.zzserver.top_error.log;
           
       location ~ [^/]\.php(/|$) {
       fastcgi_split_path_info ^(.+?\.php)(/.*)$;
       if (!-f $document_root$fastcgi_script_name) {
           return 404;
       }
   
       # Mitigate https://httpoxy.org/ vulnerabilities
       fastcgi_param HTTP_PROXY "";
   
       fastcgi_pass 127.0.0.1:9000;
       #fastcgi_pass unix:/run/php/php-fpm.sock;
       fastcgi_index index.php;
   
       # include the fastcgi_param setting
       include fastcgi_params;    
       fastcgi_param  SCRIPT_FILENAME   $document_root$fastcgi_script_name;
       fastcgi_param  PATH_INFO         $fastcgi_path_info;
       }
   
       client_max_body_size 1024m; 
       location / {
           client_max_body_size 1024m; 
       }
   }
   
   ```

   

   test and reload nginx

   ```bash
   nginx -t
   nginx -s reload
   ```



finally auto start nginx mysql and php-fpm

```bash
systemctl enable nginx
systemctl enable mysql
systemctl enable php-fpm
```



setup path

```
cd /usr/local
mkdir data && chown -R apache:apache data

```



set language

