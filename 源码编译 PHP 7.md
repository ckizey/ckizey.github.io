### 一、解决依赖

1. 安装 libxml2

    ```yum install -y libxml2 libxml2-devel```
    
    >依赖缺失时对应报错：
    configure: error: xml2-config not found. Please check your libxml2 installation.
    

2. 安装 curl

    ```yum -y install curl-devel```

    >依赖缺失时对应报错：
    configure: error: Please reinstall the libcurl distribution -
    easy.h should be in <curl-dir>/include/curl/
    
3. 安装 libmcrypt
    ```
    wget ftp://mcrypt.hellug.gr/pub/crypto/mcrypt/libmcrypt/libmcrypt-2.5.7.tar.gz
    tar -zxvf libmcrypt-2.5.7.tar.gz
    cd libmcrypt-2.5.7
    ./configure
    make && make install
    ```
    
    >缺失时对应报错：configure: error: mcrypt.h not found. Please reinstall libmcrypt.



4. 安装 gd 库

    ```yum install -y gd gd-devel```


5. 安装 jpeg 支持

    ```
    wget  http://www.ijg.org/files/jpegsrc.v9a.tar.gz
    tar zxvf jpegsrc.v9a.tar.gz
    cd jpeg-9a
    ./configure --prefix=/usr/local/jpeg
    make && make install
    ```
    
    >用于支持验证码生成
    
6. 安装 freetype
    ```
    http://www.freetype.org/download.html
    http://download.savannah.gnu.org/releases/freetype/
    wget http://download.savannah.gnu.org/releases/freetype/freetype-2.8.tar.gz

    https://sourceforge.net/projects/freetype/files/  (访问较好)
    
    tar zxvf freetype-2.8.tar.gz
    cd freetype-2.8
    ./configure --prefix=/usr/local/freetype
    make && make install
    ```
    
    >用于在生成验证码时，支持 freetype 字体验证码
    
7. 编译前配置
    ```
    ./configure \
    --prefix=/usr/local/php7 \
    --with-mysql-sock=/tmp/mysql.sock \
    --with-mysqli=shared,mysqlnd \
    --with-jpeg-dir=/usr/local/jpeg \   #需要事先安装对应组件
    --with-png-dir=/usr/local/libpng \  #若已 yum 安装 GD，此参数可忽略
    --with-freetype-dir=/usr/local/freetype \   #需要事先安装对应组件
    --with-gd \
    --enable-gd-native-ttf \
    --with-openssl \
    --with-iconv \
    --enable-shared \
    --enable-xml \
    --enable-mbstring \
    --enable-mbregex \
    --enable-ftp \
    --enable-sockets \
    --with-gettext \
    --enable-session \
    --with-curl \
    --enable-fpm \
    --with-mcrypt \
    --with-config-file-path=/usr/local/php7/etc \
    --with-apxs2=/usr/local/apache2/bin/apxs    #配置Apache的php模块
    ```

8. 集成至 Apache 时报错解决办法

    ```
    修改Apache安装目录下bin/apxs第一行Perl的二进制执行文件路径（如：#！/usr/bin/per -w）
    ```

    >Configuring SAPI modules
    checking for Apache 2.0 handler-module support via DSO through APXS... 
    Sorry, I cannot run apxs.  Possible reasons follow:
    >1. Perl is not installed
    >2. apxs was not found. Try to pass the path using --with-apxs2=/path/to/apxs
    >3. Apache was not built using --enable-so (the apxs usage page is displayed)
    
    >The output of /usr/local/apache2/bin/apxs follows:
    ./configure: /usr/local/apache2/bin/apxs: /replace/with/path/to/perl/interpreter: bad interpreter: No such file or directory
    configure: error: Aborting
    
    
### 二、编译安装

1. 编译

    ```make```
        
    对应错误原因：这是由于内存小于1G所导致
    make: *** [ext/fileinfo/libmagic/apprentice.lo] Error 1
    在./configure加上选项:
    --disable-fileinfo

2. 安装

    ``` make install ```


### 三、配置

1. 配置 php-fpm 并开启错误日志
```
cp /usr/local/php7/etc/php-fpm.conf.default  /usr/local/php7/etc/php-fpm.conf

设置运行用户
vim /usr/local/php7/etc/php-fpm.conf
;pid = run/php-fpm.pid        #将前面的注释去掉

# 找到如下配置项设置为 On
log_errors = On

# 加入如下配置错误日志路径
error_log = /usr/local/php7/logs/error.log
```

2. 配置 www.conf
````
cp /usr/local/php7/etc/php-fpm.d/www.conf.default /usr/local/php7/etc/php-fpm.d/www.conf
````

3. 配置php.ini
````
cp /usr/src/php-7.0.7/php.ini-development /usr/local/php7/etc/php.ini
````

4. 添加php-fpm用户和用户组
````
groupadd php-fpm
useradd -s /sbin/nologin -g php-fpm php-fpm
````

5. 配置php-fpm服务
````
cp /usr/src/php-7.0.7/sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
chmod +x /etc/init.d/php-fpm
chkconfig --add php-fpm
chkconfig php-fpm on
````

6. 编辑nginx配置文件，设置php解析器
````
location ~ \.php$ {
    root           html;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME   $document_root$fastcgi_script_name;
    include        fastcgi_params;
}
````
> nginx 与 php之间的通讯由 TCP 改为 Unix socket 方式：
>
> 1. 编辑 PHP 安装路径下 etc/php-fpm.d/www.conf
> 2. 将 `listen = 127.0.0.1:9000` 改为 `listen = /dev/shm/php-cgi.sock`
>   （很多教程使用路径/tmp，而路径/dev/shm是个tmpfs，速度比磁盘快得多）
> 3. 将如下代码的注释去掉，以修改 `php-cgi.sock` 的所属用户，避免权限问题
> ```
> ;listen.owner = nobody
> ;listen.group = nobody
> ;listen.mode = 0660
> ```
> 4. 编辑 Nginx 对应配置，修改 PHP 文件的解析方式为如下：
> ```
> location ~ [^/]\.php(/|$) {
>       add_header Access-Control-Allow-Origin *;
>       #fastcgi_pass   127.0.0.1:9000;
>       fastcgi_pass   unix:/dev/shm/php-cgi.sock;
>       fastcgi_index index.php;
>       include fastcgi.conf;
>       root /data/www/template/public;
> }
> ```
> 5. 重启 `nginx` 和 `php-fpm` 服务即可:
> ```
> service nginx restart
> service php-fpm restart
> ```

7. 编辑Apache配置文件，设置php解析器

- 查看已加载模块

```
/usr/local/apache/bin/apachectl -t -D DUMP_MODULES
```
    查看到有 php7_module (shared)就可以


- 编辑 httpd.conf 文件，加入如下代码

```
AddType application/x-httpd-php .php
DirectoryIndex index.php index.htm index.html
LoadModule php7_module        modules/libphp7.so
```
- 重启 Apache 服务即可

