---
layout: post
title: Linux下安装MySQL
category: 安装教程
tags: [mysql,linux]
excerpt: Linux下安装MySQL
keywords: linux,mysql,安装教程
---

[Linux下安装MySQL](https://www.jianshu.com/p/21dd4731747f)

centos7中默认安装了数据库MariaDB，如果直接安装MySQL的话，会直接覆盖掉这个数据库，当然也可以手动删除一下：

### 删除卸载mariadb

```shell
rpm -qa|grep mariadb  // 查询出来已安装的mariadb
rpm -e --nodeps 文件名  // 卸载mariadb，文件名为上述命令查询出来的文件
```

然后现在开始将当前目录切换到root也就是：    

```shell 
cd ~
```



### 下载与安装MySQL

这里采用Yum管理好了各种rpm包的依赖，能够从指定的服务器自动下载RPM包并且安装，所以在安装完成后必须要卸掉，否则会自动更新。

* **安装MySQL官方的yum repository**

  ```shell
  wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
  ```

* **下载rpm包**

  ```shell
  yum -y install mysql57-community-release-el7-10.noarch.rpm
  ```

* **安装MySQL服务**

  ```shell
  yum -y install mysql-community-server
  ```

  安装完成后会出现`complete！`

* **启动MySQL服务**

  ```shell
  systemctl start  mysqld.service
  ```

* 查看MySQL的状态

  ```shell
  systemctl status mysqld.service
  ```

![MySQL的启动状态](https://upload-images.jianshu.io/upload_images/2710833-5d261ad79a0d9f0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* MySQL常用的几个命令

  ```shell
  重启：
  systemctl restart mysqld.service
  停止：
  systemctl stop mysqld.service
  查看状态：
  systemctl status mysqld.service
  开启启动配置：
  systemctl enable mysqld
  systemctl daemon-reload   //刚刚配置的服务需要让systemctl能识别，就必须刷新配置
  ```

### 如何排查MySQL无法启动的问题？

查看mysql的配置`cat /etc/my.cnf`，其中`log-error=/var/log/mysqld.log`文件可以看到mysql的报错日志。只另打开一个窗口，输入`tail -f /var/log/mysqld.log`监听错误日志，就能根据日志定位出错误在哪。


### 首次登陆MySQL，并操作

* 登录命令，root用户登录，然后准备输入密码。

  ```shell
  mysql -u root -p
  ```

  第一次启动MySQL后，就会有临时密码，这个默认的初始密码在`/var/log/mysqld.log`文件中，我们可以用这个命令来查看：

  ```shell
  grep "password" /var/log/mysqld.log
  ```

  复制`root@localhost：`后面的就是你的首次登陆的临时密码

* 登录成功后，修改密码

  ```mysql
  mysql> use mysql;
  Database changed
  
  mysql> update mysql.user set authentication_string=password('新密码XXXX') where user='root' ;
  Query OK, 1 row affected, 1 warning (0.00 sec)
  Rows matched: 1  Changed: 1  Warnings: 1
  
  mysql> SET PASSWORD = PASSWORD('新密码XXXX');
  ```

  然后就可以用新密码登录了。

到了5.7，在部署完后，会有个默认的密码产生，如何查看：
`cat /var/log/mysqld.log | grep "temporary password"`
你使用默认密码第一次登录后，需要使用alter命令修改密码，否则什么操作也不允许。在修改默认密码的时候需要注意一下下面的坑。
修改密码的sql语句：
`alter user 'root'@'localhost' identified by 'xxx' PASSWORD EXPIRE NEVER account unlock;`

有个密码过期，你不指定，就是默认的值是`default_password_lifetime`指定的360天，需要注意下。

生成的初始密码会在日志中记录，所以对于自动化运维平台来说，多了一步处理，需要先去日志中查找初始密码，在修改密码。

  当然如果你的密码过于简单，需要再次登录后，可能会出现无法执行sql命令的情况，这时候就需要修改密码的登等级：

  ```mysql
  mysql> set global validate_password_policy=0;  //改变密码等级
  
  mysql> set global validate_password_length=4;   //改变密码最小长度
  ```

  



### MySQL的设置

* **MySQL的utf8的设置**

```mysql
character_set_server=utf8
init_connect='SET NAMES utf8'
```

采用navicat新建数据库时，需要将编码方式设置为，字符集：utf8 -- UTF-8 Unicode ，排序规则：utf8_general_ci

* **配置文件的说明**
  `/etc/my.cnf` 这是mysql的主配置文件
  `/var/lib/mysql` mysql数据库的数据库文件存放位置
  `/var/log` mysql数据库的日志输出存放位置


------
`本文是作者根据日常业务场景，写出的一些解决问题或实施想法的历程。如有错误的地方，还请指出，相互学习，共同进步。`

------


[下一篇：MySQL如何开启binlog？binlog三种模式的分析
](https://www.jianshu.com/p/8e7e288c41b1)





