[TOC]



#### 一 安装命令

```shell
 ./configure  --prefix=/usr/local/php71 --with-curl --with-freetype-dir --with-jpeg-dir --with-gd --with-gettext --with-iconv-dir --with-kerberos --with-libdir=lib64 --with-libxml-dir --with-mysqli --with-openssl --with-pcre-regex --with-pdo-mysql --with-pdo-sqlite --with-pear --with-png-dir --with-xmlrpc --with-xsl --with-zlib --enable-fpm --enable-bcmath --enable-libxml --enable-inline-optimization --enable-gd-native-ttf --enable-mbregex --enable-mbstring --enable-opcache --enable-pcntl --enable-shmop --enable-soap --enable-sockets --enable-sysvsem --enable-xml --enable-zip --enable-ftp --with-config-file-path=/usr/local/php71/etc
```



#### 二 可能的报错的以及解决方案

1. ##### centos下安装php报错：libxml2 not found. Please check your libxml2 installation

   ```shell
   yum install libxml2-devel
   ```

2. ##### centos编译安装PHP出现 cURL version 7.10.5 or later is required

   ```shell
   yum install curl-devel
   ```


3. ##### 安装php过程中的错误和解决方式 configure: error: jpeglib.h not found

   ```shell
   32 bit:
   yum install libjpeg libpng freetype libjpeg-devel libpng-devel freetype-devel -y
   
   64 bit:
   yum install libjpeg.x86_64 libpng.x86_64 freetype.x86_64 libjpeg-devel.x86_64 libpng-devel.x86_64 freetype-devel.x86_64 -y
   ```


4. ##### configure: error: xslt-config not found. Please reinstall the libxslt >= 1.1.0 distribution

  ```
  yum install -y libxslt-devel*
  ```


5. ##### phpize生成编译配置文件时：$PHP_AUTOCONF environment variable. Then, rerun this script.

   ```
   yum install autoconf
   ```

   

#### 三 需要修改的配置项

###### 3.1 拷贝配置

```shell
cp php.ini-production /etc/php.ini
cp /usr/local/php71/etc/php-fpm.conf.default /usr/local/php71/etc/php-fpm.conf
cp /usr/local/php71/etc/php-fpm.d/www.conf.default /usr/local/php71/etc/php-fpm.d/www.conf
cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
chmod +x /etc/init.d/php-fpm
```

###### 3.2 创建用户/用户组（如果需要）

```
groupadd www
useradd -g www www
```

###### 3.3 php.ini

```shell
date.timezone = PRC
error_log = /usr/local/php71/var/log/php71_error.log
upload_max_filesize = 20M
```

###### 3.4 php-fpm.d/www.conf

```shell
user = www
group = www

;listen = 127.0.0.1:9000
listen = /usr/local/php71/var/run/php-cgi.sock

listen.mode = 0666 ## 默认0660，如果不改的话，很有可能用户和用户组访问会有权限问题

pm.max_children = 1200
pm.start_servers = 20
pm.min_spare_servers = 20
pm.max_spare_servers = 100

request_terminate_timeout = 300

slowlog = /usr/local/php71/var/log/php_slow.log
```



#### 附1 php.ini示例

#### 附2 www.ini示例

#### 附3 nginx.conf示例

