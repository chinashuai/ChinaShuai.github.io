---
layout: post
title: linux安装nginx
category: 安装教程
tags: [nginx]
excerpt: linux安装nginx
keywords: linux,nginx,安装教程
---

安装环境：

> CentOS release 6.8 (Final)

1.安装PCRE库

    $ cd /usr/local/
    $ wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.39.tar.gz 
    $ tar -zxvf pcre-8.39.tar.gz
    $ cd pcre-8.39
    $ ./configure
    $ make
    $ make install

2.安装zlib库

    $ cd /usr/local/ 
    $ wget http://zlib.net/zlib-1.2.11.tar.gz
    $ tar -zxvf zlib-1.2.11.tar.gz
    $ cd zlib-1.2.11
    $ ./configure
    $ make
    $ make install

3.安装ssl

    $ cd /usr/local/
    $ wget http://www.openssl.org/source/openssl-1.0.1j.tar.gz
    $ tar -zxvf openssl-1.0.1j.tar.gz
    $ cd openssl-1.0.1j
    $ ./config
    $ make
    $ make install

4.安装nginx

    $ cd /usr/local/
    $ wget http://nginx.org/download/nginx-1.8.0.tar.gz
    $ tar -zxvf nginx-1.8.0.tar.gz
    $ cd nginx-1.8.0  
    $ ./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/usr/local/pcre-8.39 --with-zlib=/usr/local/zlib-1.2.11
    $ make
    $ make install

#####提示：

我的一位朋友系统版本为 `CentOs 7 `， 用相同的流程安装 nginx 报出下面的异常，

```
checking for OpenSSL library ... not found  
  
./configure: error: SSL modules require the OpenSSL library.
You can either do not enable the modules, or install the OpenSSL library
into the system, or build the OpenSSL library statically from the source
with nginx by using --with-openssl=<path> option. 
```

提示安装时没有发现 `--with-openssl=<path> option` 的路径配置，将 安装 `nginx` 的 `./configure …` 指令中 `--with-http_ssl_module`      改为      `--with-openssl=/usr/local/ssl`  , 具体路径，请自行查看。


    checking for OpenSSL library ... not found
    
    ./configure: error: SSL modules require the OpenSSL library.
    You can either do not enable the modules, or install the OpenSSL library
    into the system, or build the OpenSSL library statically from the source
    with nginx by using --with-openssl=<path> option. 





5.启动

    $ /usr/local/nginx/sbin/nginx

6.是否启动成功

浏览器输入： ip:80 出现如下所示网页则成功





配置文件位于：

    $ /usr/local/nginx/conf/nginx.conf

Nginx配置文件常见结构的从外到内依次是「http」「server」「location」等等，缺省的继承关系是从外到内，也就是说内层块会自动获取外层块的值作为缺省值。



Server

接收请求的服务器需要将不同的请求按规则转发到不同的后端服务器上，在 nginx 中我们可以通过构建虚拟主机（server）的概念来将这些不同的服务配置隔离。

    server {
        listen       80;
        server_name  localhost;
        root   html;
        index  index.html index.htm;
    }









常用指令：

    $ /usr/local/webserver/nginx/sbin/nginx -s reload  //重新载入配置文件
    $ /usr/local/webserver/nginx/sbin/nginx -s reopen // 重启 Nginx
    $ /usr/local/webserver/nginx/sbin/nginx -s stop //停止 Nginx






