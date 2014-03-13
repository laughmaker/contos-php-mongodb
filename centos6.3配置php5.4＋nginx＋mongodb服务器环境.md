# CentOS6.3配置PHP5.4＋NGINX＋Mongodb服务器环境

首先系统安装完成后，执行：`yum -y update` 命令，升级补丁。

---

## 安装PHP5.4

[参考教程](http://www.iitshare.com/centeros-6-3-64-bit-install-php-5-4-3.html)

**1.安装支持套件**

yum -y install gcc gcc-c++ autoconf libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel zlib zlib-devel glibc glibc-devel glib2 glib2-devel bzip2 bzip2-devel ncurses ncurses-devel curl curl-devel e2fsprogs e2fsprogs-devel krb5 krb5-devel libidn libidn-devel openssl openssl-devel openldap openldap-devel nss_ldap openldap-clients openldap-servers

**2.安装支持库**

*  [pcre-8.30.tar.gz下载](http://pan.baidu.com/share/link?shareid=2787262895&uk=553700327&fid=2190701233)
* [libiconv-1.14.tar.gz下载](http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.14.tar.gz)
*  [libmcrypt-2.5.8.tar.gz下载](http://centos.googlecode.com/files/libmcrypt-2.5.8.tar.gz)
*  [mhash-0.9.9.9.tar.gz下载](http://nchc.dl.sourceforge.net/project/mhash/mhash/0.9.9.9/mhash-0.9.9.9.tar.gz)
*  [mcrypt-2.6.8.tar.gz下载](http://nchc.dl.sourceforge.net/project/mcrypt/MCrypt/2.6.8/mcrypt-2.6.8.tar.gz)

		安装方法：
		tar zxvf 文件名.tar.gz
		cd 文件夹
		./configure
		make make install
		
**备注：**

	配置mcrypt-2.6.8.tar.gz支持库时出现以下错误
	configure: error: *** libmcrypt was not found
	解决方法
	运行 export LD_LIBRARY_PATH=/usr/local/lib: LD_LIBRARY_PATH
	然后编译执行
	
	# ./configure
	# make
	# make install
	
**3.安装PHP5.4**

**[下载地址](http://am1.php.net/get/php-5.4.26.tar.gz/from/this/mirror)**

**安装方法：**

	tar zvxf php-5.4.26.tar.gz
	cd php-5.4.26
	./configure -prefix=/usr/local/php -with-config-file-path=/usr/local/php/etc -with-iconv-dir=/usr/local/lib -with-freetype-dir -with-jpeg-dir -with-png-dir -with-zlib -with-libxml-dir=/usr -enable-xml -disable-rpath -enable-bcmath -enable-inline-optimization -with-curl -with-curlwrappers -enable-fpm -enable-mbstring -with-mcrypt -with-gd -enable-gd-native-ttf -with-openssl -with-mhash -enable-pcntl -enable-sockets -with-xmlrpc -enable-soap -without-pear -with-fpm-user=www -with-fpm-group=www --disable-fileinfo
	make
	make install
	
**备注：**

	若./configure...时，报：`configure: error: Cannot find ldap libraries in /usr/lib`错误，则执行：cp -frp /usr/lib64/libldap* /usr/lib/ 命令，再./configure....
	若./configure...时，报`make: *** [sapi/cli/php] Error 1`错误，则执行：ZEND_EXTRA_LIBS='-liconv' 命令，再make install
	
**添加配置文件：**

在安装文件目录，执行以下命令：

	* cp php.ini-production /usr/local/php/lib/php.ini
	* cp php.ini-production /usr/local/php/etc/php.ini
	* cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
	
编辑php-fpm.conf文件为以下内容：

	[global]
	pid = run/php-fpm.pid
	error_log = log/php-fpm.log
	log_level = notice
	[www]
	listen = 127.0.0.1:9000
	user = www
	group = www
	pm = dynamic
	pm.max_children = 5
	pm.start_servers = 2
	pm.min_spare_servers = 1
	pm.max_spare_servers = 3
	pm.max_requests = 500
	
测试php-fpm配置：

	/usr/local/php/sbin/php-fpm -t
	如果显示以下信息，则说明安装成功
	NOTICE: configuration file /usr/local/php/etc/php-fpm.conf test is successful

启动9000端口号

	防火墙中开启php默认的端口号9000，如果服务器没有开启防火墙，则不需要
	
启动php-fpm：

	/usr/local/php/sbin/php-fpm
	如果启动报错：ERROR: [pool www] cannot get uid for user ‘www’
	增加用户即可,具体的代码：useradd www -M -s /sbin/nologin


－－－

## 安装NGINX

[下载地址](http://nginx.org/packages/centos/6/x86_64/RPMS/nginx-1.4.6-1.el6.ngx.x86_64.rpm)

[nginx官方源库](http://nginx.org/packages/centos/6/x86_64/RPMS/nginx-1.4.6-1.el6.ngx.x86_64.rpm)：可以找寻其他版本

安装：

	rpm -ivh nginx-release-centos-6-0.el6.ngx.noarch.rpm


---

## 安装Mongodb

**1.添加yum源：**

	vi /etc/yum.repos.d/10gen.repo
	添加以下语句：
	
	[10gen] 
	name=10gen Repository 
	baseurl=http://downloads-distro.mongodb.org/repo/redhat/os/x86_64 
	gpgcheck=0 

**2.安装MongoDB的服务器端和客户端工具：**

	yum install mongo-10gen-server
	yum install mongo-10gen
	
3.更新Mongodb：
	
	停掉mongodb，执行
	yum update mongo-10gen mongo-10gen-server
	
启动Mongodb：service mongod start


---

### 以上将三个库都安装完成了，下面需要装三个关联起来。


##### 首先配置NGINX，支持PHP解析

1.打开/etc/nginx/conf.d目录下的default.conf文件。
2.如下代码所示，将`location` 中的相关路径保持一致，差添加index.php解析

	server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.php index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        root           /usr/share/nginx/html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /usr/share/nginx/html$fastcgi_script_name;
        include        fastcgi_params;
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}


##### 安装PHP对Mongodb的驱动

1.**[下载驱动]**(http://github.com/mongodb/mongo-php-driver)

2.**解压，并进入到该目录。**

3.执行以下命令：

	phpize		或者  /usr/local/php/bin/phpize 
	./configure	或者  ./configure  --with-php-config=/usr/local/php/bin/php-config --with-mongodb 
	make all
	sudo make install
	
4.编辑php配置文件，添加对mongodb的支持
	
	vim /usr/local/php/etc/php.ini
	
	在最后添加如下代码：
	
	extension=mongo.so
	
	保存退出
	
	
##### 安装MongoDB Administrator

1.[下载地址](http://rockmongo.com/downloads)

2.将文件解压到 `/usr/share/nginx/html` 文件夹里

3.在浏览器里打开：http://localhost/rockmongo/index.php

4.初始密码：admin admin

5.若添加管理员后，去除默认管理员，需要打开rockmogo里的配置文件，将其中一行改为：

	vim /usr/share/nginx/html/rockmongo/config.php
	
	$MONGO["servers"][$i]["mongo_auth"] = true;//enable mongo authentication?


---
---


至此，已全部配置完毕，在配置完成后，需要重新启动服务。

**以下是几个常用命令**

	* 启动NGINX：service nginx start
	* 停步NGINX：service nginx stop
	* 重启NGINX：service nginx reload
	
	
	* 启动PHP： /usr/local/php/sbin/php-fpm
	* 停止PHP：killall -9 php-fpm
	
	* 启动Mongo：service mongos start
	* 停止Mongo：service mongos stop

**相关配置文件所在目录：**

	* nginx 配置文件目录：/etc/nginx/*.conf
	* php配置文件目录：/usr/local/php/etc/php.ini
	* 网页文件目录：/usr/share/nginx/html 
		















