---
title: MySQL数据库部署与操作
date: 2023-09-21 21:06:01
copyright: true
tags: [互联网应用开发实践, MySQL, SpringBoot]
categories:
- 技术笔记
- 互联网应用开发实践
---



本次的实验环境是`CentOS 7.9` 已经安装了宝塔面板，并安装了`Docker管理器3.9.1`。

<!--more-->


---



# MySQL的安装与部署



## 1. 在Docker中安装MySQL

**【将Docker中MySQL数据库挂载到服务器】**
将Docker中的数据以及配置文件映射到服务器的文件系统上，这样当删除了Docker容器后，之前的数据记录依然存在。
```shell
docker run --restart=always --privileged=true -d \
-v /dockerImageFile/mysql/data:/var/lib/mysql \
-v /dockerImageFile/mysql/conf:/etc/mysql/conf.d \
-v /dockerImageFile/mysql/my.cnf:/etc/mysql/my.cnf 
-p 3306:3306 \
--name mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
mysql:5.7
```

- `-v`：文件挂载（`宿主机文件路径`:`Docker容器文件路径`）；
- `-p`：端口映射（`对外暴露端口`:`Docker容器内部端口`）；
- `--name`：容器名称；
-  `MYSQL_ROOT_PASSWORD=123456`：设置MySQL数据库的初始root密码为`123456`；
- `mysql:5.7`：指定MySQL版本为5.7。

**【若不需要挂载的话可以正常运行下面的命令】**
```shell
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
```


**【注意】**：上述指令只适用于Linux、Windows系统等x86架构芯片的机器，若为M1芯片，需要使用如下命令安装较新的MySQL版本。

```shell
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql/mysql-server
```

### 可能会遇到的错误
#### ※ 数据库管理工具连接错误
**错误提示**：`Host '172.17.0.1' is not allowed to connect to this MySQL server`

说明所连接的用户帐号没有远程连接的权限，只能在本机（localhost）登录，需更改 MySQL 数据库里的 user 表里的 host 项。

- 进入容器：
```sql
docker exec -it mysql /bin/bash
```


- 登录MySQL：
```sql
mysql -uroot -p123456
```

- 修改权限表格
```sql
use mysql;
update user set host = '%' where user = "root";
flush privileges;
```
#### ※ 无修改权限
**错误提示**：`Access denied for user 'dorm'@'%to database 'xxx'`

```sql
grant all privileges on *.* to 'dorm'@'%';
```



## 2. 在Docker中安装phpMyAdmin

phpMyAdmin 是一种 MySQL图形化 管理工具，该工具是基于 Web 跨平台的管理程序，并且支持简体中文。在宿主机的命令行运行下面的代码即可安装。
```shell
docker run --name phpmyadmin -p 8080:80 --link mysql:db -d phpmyadmin/phpmyadmin:latest
```
- `phpmyadmin`：容器名称；
- `8080:80`：开放端口 8080 来映射宿主机的 80 端口；
- `mysql`：刚才设置的MySQL的容器名称，如果刚才修改容器名称的话，一定记得要在这里修改。

安装完成后，通过 `公网IP:8080` 即可访问如下页面，初始账号为 root，密码为刚才设置的`MYSQL_ROOT_PASSWORD=`后面的内容（`123456`）
![PHPMyAdmin](/images/MySQL数据库部署与操作/PHPMyAdmin.png)

### 可能会遇到的问题
#### ※ 无法访问phpMyAdmin
这种情况大多数是因为防火墙（或服务器的安全组规则）导致的。因为刚才自定义的 `8080` 端口在服务器中是默认不开放的，所以要配置服务器的安全组，将其开放。
![腾讯云开放安全组](/images/MySQL数据库部署与操作/腾讯云开放安全组.png)




## 3. 创建账户与数据库

 新建一个项目账户（dorm），如下图所示。
 ![新增项目用户](/images/MySQL数据库部署与操作/新增项目用户.png)


**【注意】：在项目中尽量不要使用root用户管理项目，而是单独创建以该项目名称为命名的数据库账户，并建立同名的数据库进行管理。**



## 4. 本地MySQL的安装与操作

本人使用的是 M1 的 Mac 环境，因此对应安装了 M1 版本的 Docker 和 MySQL，具体步骤和①两步类似，所以这里就不再赘述。
本地是数据库管理软件我使用的的是免费开源的数据库管理软件“Sequel Ace”，它具有数据库管理软件的基本功能，在AppStore中就可以搜索到，而且是完全免费的。
![Sequel Ace](/images/MySQL数据库部署与操作/Sequel_Ace.png)

![设计数据库](/images/MySQL数据库部署与操作/设计数据库.png)


---





# SpringBoot 连接 MySQL



## 1. porm.xml 文件

在 porm.xml 文件中的`<dependencies>`中添加 JDBC 的依赖。
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```
更新 Maven 依赖。



## 2. application.yml 文件

修改SpringBoot配置文件，application.yml 文件，增加数据库相关信息。
```java
server:  
  port: 8090
spring:
  datasource:
	# url: jdbc:mysql://127.0.0.1:3306/DORM  # 本地测试环境
    url: jdbc:mysql://42.194.xxx.xx:3306/dorm	# 服务器环境
    username: dorm
    password: dorm
    driver-class-name: com.mysql.cj.jdbc.Driver
```



## 3. Java类文件

添加Java类文件，DormInfo.java。
```java
package com.songxj.dormitory_system.api;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

import java.util.List;
import java.util.Map;


@Controller
public class DormInfo {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @RequestMapping(value = "/dormInfo", method = RequestMethod.GET)
    @ResponseBody
    public List<Map<String, Object>> getDormInfo() {
        String sql = "select * from dorm"; // SQL查询语句
        return jdbcTemplate.queryForList(sql);
    }
}
```



## 4. 本地测试

本地测试结果如下：
![本地测试](/images/MySQL数据库部署与操作/本地测试.png)




## 5. 打包 jar 包
![打jar包](/images/MySQL数据库部署与操作/打jar包.png)




## 6. 本地开发环境连接服务器Docker


连接Docker宿主机，修改Docker服务文件

```shell
vi  /lib/systemd/system/docker.service
```

将原来的`ExecStart`前面加上`#`号注释掉，然后在下面追加一行
```shell
ExecStart=/usr/bin/dockerd    -H tcp://0.0.0.0:2375    -H unix:///var/run/docker.sock
```
重新加载配置
```shell
systemctl daemon-reload
```

重启Docker服务
```shell
systemctl restart docker.service
```

开放宿主机安全组中的 2375 端口
![开放宿主机安全组中的2375端口](/images/MySQL数据库部署与操作/开放宿主机安全组中的2375端口.png)



在IDEA中安装Docker插件。
![Docker插件](/images/MySQL数据库部署与操作/Docker插件.png)


在Docker插件中连接宿主机服务器
![配置Docker](/images/MySQL数据库部署与操作/配置Docker.png)




## 7. Docker 在本地创建（build）并运行（run）镜像

- 创建镜像
```shell
docker build -t dorm_sys:1.0.0 . 
```

- 运行镜像
```shell
docker run -d -p 10110:8090 --name dorm_sys dorm_sys:1.0.0
```

**【M1芯片打包的jar包在Linux环境无法运行】**
- M1芯片（arm64）构建Linux（amd64）可运行的镜像
```shell
docker build --platform linux/amd64 -t dorm_sys:1.0.0 . 
```



## 8. 上传本地镜像到阿里云

首先在阿里云免费申请容器镜像服务（[https://cr.console.aliyun.com/repository/](https://cr.console.aliyun.com/repository/)）

- 登录阿里云容器镜像服务
```shell
docker login --username=songxj01 registry.cn-hangzhou.aliyuncs.com
```

- 上传镜像
```shell
docker tag dorm_sys:1.0.0 registry.cn-hangzhou.aliyuncs.com/songxj01/dorm_sys:1.0.0
docker push registry.cn-hangzhou.aliyuncs.com/songxj01/dorm_sys:1.0.0
```



## 9. 在服务器拉取镜像并运行

```shell
docker pull registry.cn-hangzhou.aliyuncs.com/songxj01/dorm_sys:1.0.0
```

```shell
docker run -d -p 10110:8090 --name dorm_sys registry.cn-hangzhou.aliyuncs.com/songxj01/dorm_sys:1.0.0
```

在线测试：
![在线测试](/images/MySQL数据库部署与操作/在线测试.png)

<br/><br/><br/><br/>

