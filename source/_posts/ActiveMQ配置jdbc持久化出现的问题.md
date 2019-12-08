---
title: ActiveMQ配置jdbc持久化出现的问题
date: 2019-08-26 16:07:28
tags:
- ActiveMQ
categories:
- JAVA
---





# ActiveMQ配置jdbc持久化出现的问题

当我配置activemq.xml，重启时发现无进程运行，又不报错

{% asset_img start.jpg %}

查看日志 在 /data/activemq.log下

<!--more-->

**tail -f activemq.log**

发现报错误了

 **java.lang.IllegalStateException: BeanFactory not initialized or already closed - call 'refresh' before accessing beans via the ApplicationContext**

{% asset_img log_error.jpg %}

网上的主流解决方式：

1.确认计算机主机名名称没有下划线；

2.如果是win7，停止ICS(运行-->services.msc找到Internet Connection Sharing (ICS)服务,改成手动启动或禁用)

重新启动activeMQ即可。



但我发现我的windows主机名是英文加数字没有什么问题， ICS服务也是禁用的。

难道是 centos7的主机名问题？

我将自己的主机名改成了纯英文名 eiletxie

发现

**这里说明：**

如果有要虚拟机重启几次才可以开启activemq的情况是需要修改主机名的，我这里的不是这种原因。

```shell
[root@centos7 ~]$ hostnamectl set-hostname eiletxie          # 使用这个命令会立即生效且重启也生效
[root@centos7 ~]$ hostname                                                 # 查看下
eiletxie 
[root@centos7 ~]$ vim /etc/hosts                                           # 编辑下hosts文件， 给127.0.0.1添加hostname
[root@centos7 ~]$ cat /etc/hosts                                           # 检查
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 eiletxie 
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

------

突然想了一下，是不是防火墙的原因。

本地ping的activemq服务器，发现没问题。

在服务器上ping 了一下本地的IP地址

果然我本地的防火墙没关，导致没法访问。

关闭防火墙后再次启动，还是无法打开网页，查看日志

{% asset_img root_error.jpg %}



**这里已经说** **Host is not allowed to connect to this MySQL server**

MySQL不允许root用户远程登录，所以远程登录失败了，解决方法如下：

1. ***在装有MySQL的机器上登录MySQL mysql -u root -p密码***
2. ***执行******use mysql;***
3. ***执行******update user set host = '%' where user = 'root';***
4. ***执行******FLUSH PRIVILEGES; 刷新权限配置***

***先说明： 这种方式是开放所有IP的root访问权限。如果是正式环境还是建议用 开放指定IP的方式***

***这里有两种直接设置的方式***

*1. 授权用户root使用密码jb51从任意主机连接到*[*mysql*](https://www.baidu.com/s?wd=mysql&tn=44039180_cpr&fenlei=mv6quAkxTZn0IZRqIHckPjm4nH00T1Y3PjN9PhRLmH99PymzuycY0ZwV5Hcvrjm3rH6sPfKWUMw85HfYnjn4nH6sgvPsT6KdThsqpZwYTjCEQLGCpyw9Uz4Bmy-bIi4WUvYETgN-TLwGUv3Erjb3PWcknWf)*服务器：*

*代码如下:*

```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'jb51' WITH GRANT OPTION;

flush privileges;
```

 

2.授权用户root使用密码jb51从指定ip为218.12.50.60的主机连接到mysql服务器：

*代码如下:*

```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@218.12.50.60'%' IDENTIFIED BY 'jb51' WITH GRANT OPTION;
flush privileges;
```



*我是本地mysql做测试用的，就直接授予所有IP访问权限了*

{% asset_img mysql_test.jpg %}

测试一下发送信息没问题



------

再查看本地mysql的 activemq 数据库，已经有核心的3张表了

{% asset_img mysql.jpg %}

未消费的message存于 activemq_msgs表中

{% asset_img mysql_msg.jpg %}